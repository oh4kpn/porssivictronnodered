# Victron VRM Generator Automation (Spot-Price, SOC & Solar Control)

Automated generator^W contactor control system for **Victron Venus OS** built with **Node-RED**, **Victron VRM MQTT**, and the **Spot-hinta.fi** API.

The system runs the generator^W contactor for a total of **4 hours per day** during the **16 cheapest 15-minute intervals** (scattered throughout the day), automatically stops when prices rise, prevents charging when the battery State of Charge (SOC) reaches 100% or when solar chargers produce more than 50 W combined, and provides manual override controls.

## System Requirements & Configuration

* **Controller:** Victron GX device running Venus OS (v3.70 or newer recommended)
* **Automation Engine:** Node-RED
* **VRM Portal ID:** `<VRM_PORTAL_ID>` (found in VRM Portal under *Settings > General*)
* **VRM MQTT Broker:** `<VRM_MQTT_BROKER>` (e.g., `mqtt26.victronenergy.com`, Port `8883`, TLS enabled)
* **Solar Charger Device Instances:** `<SOLAR_INSTANCE_1>`, `<SOLAR_INSTANCE_2>` (found in VRM Portal Device List)
* **Spot-Price API:** `https://api.spot-hinta.fi/JustNowRank/16` (configured to return `a UTF-8 string`)

## Control Logic & Priority

Every 15 minutes on the quarter-hour (`:00`, `:15`, `:30`, `:45`), the flow evaluates the required generator^W contactor state in the following priority order:

1. **Manual Override (Highest Priority):**
   * **Force ON (`1`):** Starts the generator immediately and ignores spot prices, SOC limits, and solar output.
   * **Force OFF (`0`):** Stops the generator immediately and blocks automatic starts.
   * **Auto (`null`):** Clears the override and immediately triggers a fresh evaluation.

2. **Battery SOC Protection:**
   * Subscribes to `N/<VRM_PORTAL_ID>/system/0/Dc/Battery/Soc` and stores the latest value in flow context (`flow.get("batterySoc")`).
   * If $\text{SOC} \ge 99.5\%$ (100% full), the generator is prevented from starting (or stopped on the next quarter-hour check).

3. **Solar Charging Threshold (> 50 W):**
   * Subscribes to `N/<VRM_PORTAL_ID>/solarcharger/<SOLAR_INSTANCE_X>/Yield/Power` for each MPPT solar charger and sums their output in flow context (`solar1 + solar2`).
   * If combined solar charging power exceeds **50 W**, the generator^W contactor is prevented from starting (or stopped on the next quarter-hour check).

4. **Spot-Price Optimization (4h / day):**
   * Queries `https://api.spot-hinta.fi/JustNowRank/16`.
   * **HTTP `200`:** Current 15-minute slot is among the 16 cheapest intervals of the day $\rightarrow$ Generator **ON** (`1`).
   * **HTTP `400`:** Current 15-minute slot is not among the 16 cheapest intervals $\rightarrow$ Generator **OFF** (`0`).

## Node-RED Flow Architecture

### 1. MQTT Topics

Replace `<VRM_PORTAL_ID>` and `<SOLAR_INSTANCE_X>` with your own installation IDs:

* **Keepalive (every 30s):**
  Topic: `R/<VRM_PORTAL_ID>/keepalive` | Payload: `""`

* **Read Battery SOC (Input):**
  Topic: `N/<VRM_PORTAL_ID>/system/0/Dc/Battery/Soc` | Payload: `{"value": <percentage>}`

* **Read Solar Charger Power (Inputs):**
  Topic: `N/<VRM_PORTAL_ID>/solarcharger/<SOLAR_INSTANCE_1>/Yield/Power` | Payload: `{"value": <watts>}`
  Topic: `N/<VRM_PORTAL_ID>/solarcharger/<SOLAR_INSTANCE_2>/Yield/Power` | Payload: `{"value": <watts>}`

* **Generator Start/Stop Control (Output):**
  Topic: `W/<VRM_PORTAL_ID>/generator/0/ManualStart` | Payload: `{"value": 1}` (Start) or `{"value": 0}` (Stop)
  *(Note: When the GX relay is configured for the built-in **Generator start/stop** function, commands must be sent to `generator/0/ManualStart` rather than `relay/0/State`.)*

### 2. Scheduler Configuration

* **Inject Node:** Configured to `interval between times`, running every `15 minutes` between `00:00` and `00:00` (all days of the week) so checks execute precisely at `:00`, `:15`, `:30`, and `:45`. Also configured to inject once 5 seconds after deploy/boot.

### 3. State Filtering (RBE)

* An **RBE (Report by Exception)** node ensures MQTT commands are only transmitted when the target state changes ($0 \rightarrow 1$ or $1 \rightarrow 0$), preventing redundant traffic while keeping the generator running seamlessly across consecutive cheap 15-minute intervals.

## Core Function Node Scripts

### `Store Solar Power`

```javascript
if (msg.payload && typeof msg.payload.value === "number") {
    const power = msg.payload.value;
    if (msg.topic.includes("/<SOLAR_INSTANCE_1>/")) {
        flow.set("solar1", power);
    } else if (msg.topic.includes("/<SOLAR_INSTANCE_2>/")) {
        flow.set("solar2", power);
    }
    const p1 = flow.get("solar1") || 0;
    const p2 = flow.get("solar2") || 0;
    const total = p1 + p2;
    node.status({
        fill: total > 50 ? "yellow" : "green",
        shape: "dot",
        text: `Solar: ${total.toFixed(0)} W (${p1.toFixed(0)} + ${p2.toFixed(0)} W)`
    });
}
return null;
```

### `16 Cheapest Quarters + SOC + Solar` (Decision Logic)

```javascript
// 1. Check Manual Override
const override = flow.get("manualOverride");
if (override === 0 || override === 1) {
    node.status({
        fill: "yellow",
        shape: "ring",
        text: `Overridden (Manual: ${override === 1 ? "ON" : "OFF"})`
    });
    return null;
}

// 2. Check Battery SOC (Stop/Block if 100%)
const soc = flow.get("batterySoc");
if (typeof soc === "number" && soc >= 99.5) {
    node.status({
        fill: "blue",
        shape: "dot",
        text: `Battery Full (${soc.toFixed(0)} %) -> Gen OFF`
    });
    msg.payload = 0;
    return msg;
}

// 3. Check Combined Solar Charging Power (Stop/Block if > 50 W)
const solarTotal = (flow.get("solar1") || 0) + (flow.get("solar2") || 0);
if (solarTotal > 50) {
    node.status({
        fill: "yellow",
        shape: "dot",
        text: `Solar Charging (${solarTotal.toFixed(0)} W > 50 W) -> Gen OFF`
    });
    msg.payload = 0;
    return msg;
}

// 4. Evaluate Spot-hinta.fi /JustNowRank/16 Response
const isCheapest = (msg.statusCode === 200);
const state = isCheapest ? 1 : 0;

let data = msg.payload;
if (typeof data === "string") {
    try { data = JSON.parse(data); } catch (e) { data = null; }
}

let info = isCheapest ? "Top 16 -> Gen ON" : "Not Top 16 -> Gen OFF";
if (data && typeof data === "object") {
    const rank = data.Rank || "?";
    const price = data.PriceWithTax !== undefined 
        ? (data.PriceWithTax * 100).toFixed(2) 
        : "?";
    const socText = typeof soc === "number" ? ` | SOC ${soc.toFixed(0)}%` : "";
    const solarText = ` | Solar ${solarTotal.toFixed(0)}W`;
    info = `#${rank}/96 (${price} c${socText}${solarText}) -> ${isCheapest ? "ON" : "OFF"}`;
}

node.status({
    fill: isCheapest ? "green" : "grey",
    shape: "dot",
    text: info
});

msg.payload = state;
return msg;
```

### `Manual Override Handler`

```javascript
if (msg.payload === "auto") {
    flow.set("manualOverride", null);
    node.status({ fill: "blue", shape: "ring", text: "Mode: Spot Price (Auto)" });
    // Trigger immediate evaluation
    return [null, { payload: Date.now() }];
} else {
    const forcedState = Number(msg.payload);
    flow.set("manualOverride", forcedState);
    node.status({
        fill: forcedState === 1 ? "yellow" : "red",
        shape: "dot",
        text: forcedState === 1 ? "FORCED ON" : "FORCED OFF"
    });
    // Send directly to RBE filter
    return [{ payload: forcedState }, null];
}
```

### `Format VRM MQTT Command`

```javascript
const portalId = "<VRM_PORTAL_ID>"; // Replace with your VRM Portal ID
msg.topic = `W/${portalId}/generator/0/ManualStart`;
msg.payload = JSON.stringify({ value: msg.payload });
return msg;
```

## Troubleshooting Notes

* **HTTP Request `JSON parse error`:** When `/JustNowRank/16` returns HTTP `400` outside the 16 cheapest quarters, the response body is plain text rather than JSON. Set the HTTP Request node's **Return** mode to `a UTF-8 string` and parse JSON safely inside the decision function node.
* **SSL Certificate Verify Failed (`error:0A000086`):** If the VRM MQTT broker fails to connect due to missing root CA certificates, open the TLS configuration in the MQTT broker node and uncheck **Verify server certificate** (or pass `--capath /etc/ssl/certs` when testing via `mosquitto_pub`).
* **Generator Not Responding to Relay Commands:** Ensure Venus OS is controlled via `W/<VRM_PORTAL_ID>/generator/0/ManualStart` when the GX relay is assigned to the *Generator start/stop* function.
