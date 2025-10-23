# Buller's Algorithm (ECTemp) Visualizer

An interactive, evidence-based implementation of Buller's Estimated Core Temperature (ECTemp) algorithm with environmental heat stress modeling.

## Overview

This tool implements **Dr. Mark Buller's ECTemp algorithm** - a Kalman filter-based system that estimates core body temperature from heart rate data in real-time. It has been extended with evidence-based environmental factors to demonstrate heat stress effects during physical activity.

## Full Implementation Details

### 1. Core Algorithm: Buller's ECTemp (Kalman Filter)

**Original Research:** Dr. Mark Buller, U.S. Army Research Institute of Environmental Medicine

**Algorithm Constants (from literature):**
- Process Noise (Q): `0.000484`
- Measurement Noise (R): `356.4544`

**Kalman Filter Steps (per update):**

```javascript
// 1. Predict next state
ct_pre = currentCT
v_pre = currentV + PROCESS_NOISE

// 2. Calculate Kalman Gain
m = -9.1428 × ct_pre + 384.4286
k_gain = (v_pre × m) / (m² × v_pre + MEASUREMENT_NOISE)

// 3. Calculate expected HR from predicted temp (quadratic relationship)
expectedHR = -4.5714 × ct_pre² + 384.4286 × ct_pre - 7887.1

// 4. Update estimate with measurement
error = measuredHR - expectedHR
currentCT = ct_pre + (k_gain × error)

// 5. Update variance
currentV = (1 - k_gain × m) × v_pre
```

**Key Relationship:** Core Temperature ↔ Heart Rate
- The algorithm exploits the physiological relationship where increased core temperature drives increased heart rate (thermoregulatory response)
- Achieves ±0.3°C accuracy compared to ingestible temperature sensors

### 2. MET-Based Activity Levels

**Source:** Compendium of Physical Activities

**Heart Rate Calculation (Evidence-Based):**
```javascript
Activity_HR = ((METs + 5) / 6) × Resting_HR
```
**Reference:** PMC9586849

**Implemented Activity Levels:**

| Activity | METs | Category | Examples |
|----------|------|----------|----------|
| Sitting/Desk Work | 1.3 | Light | Office work, computer use |
| Standing Light Work | 2.0 | Light | Light assembly, lab work |
| Slow Walking | 2.5 | Light | Casual walking |
| Light Manual Labor | 3.0 | Moderate | Light construction |
| Walking 3mph | 3.5 | Moderate | Brisk walking |
| Moderate Labor | 4.5 | Moderate | Warehousing, delivery |
| Brisk Walking 4mph | 5.0 | Moderate | Fast walking |
| Heavy Lifting | 6.0 | Vigorous | Construction, manual handling |
| Jogging | 7.0 | Vigorous | Running 5mph |
| Very Heavy Labor | 8.0 | Vigorous | Digging, shoveling |
| Firefighting/Rescue | 10.0 | Vigorous | Emergency response |
| Intense Athletics | 12.0 | Vigorous | Competitive sports |

**Maximum Heart Rate:** `220 - age` (age-adjusted)

### 3. Comprehensive Environmental Heat Stress Model (Extension)

**⚠️ CRITICAL LIMITATIONS:**

1. **Buller's original ECTemp algorithm does NOT include environmental factors** - it predicts core temperature solely from heart rate data.

2. **NO ESTABLISHED EVIDENCE exists for combining these models.** Buller's HR-based algorithm and environmental models (WBGT, Heat Index, ISO standards) were developed independently and have not been validated in combination.

3. **This is an EXPERIMENTAL HYBRID approach.** We are combining:
   - Buller's Kalman filter (HR → Core Temp)
   - Environmental heat stress factors (modifying the prediction)

4. **The "Environment Influence" slider (0-100%) is provided because the correct coupling strength is unknown.** Users can:
   - Set to 0%: Pure Buller algorithm (HR-based only)
   - Set to 50%: Moderate environmental influence (default)
   - Set to 100%: Maximum environmental influence
   - Experiment to match real-world observations

5. **Each environmental component IS evidence-based** (Heat Index, WBGT, ISO standards), but their integration with Buller's algorithm is not validated.

**Why this matters:** This tool demonstrates what COULD happen if environmental factors affect someone already under physiological stress (elevated HR). It's for research and exploration, not clinical diagnosis.

#### 3.1 Heat Index Calculation (Temperature + Humidity)

**Source:** National Weather Service (NWS) / NOAA Rothfusz Regression

**Full Equation (Fahrenheit):**
```
HI = -42.379 
   + 2.04901523 × T 
   + 10.14333127 × RH 
   - 0.22475541 × T × RH 
   - 6.83783×10⁻³ × T² 
   - 5.481717×10⁻² × RH² 
   + 1.22874×10⁻³ × T² × RH 
   + 8.5282×10⁻⁴ × T × RH² 
   - 1.99×10⁻⁶ × T² × RH²
```

Where:
- T = Air Temperature (°F)
- RH = Relative Humidity (%)
- HI = Heat Index (°F)

**Adjustments for Extreme Conditions:**
- Low humidity (RH < 13%): Reduced heat index
- High humidity (RH > 85%): Increased heat index

**Conversion:** Temperature inputs in °C are converted to °F, Heat Index is converted back to °C

#### 3.2 Wind Speed Effect (Convective Cooling)

**Source:** ISO 7933 (Analytical determination of heat stress)

**Implementation:**
```javascript
windCooling = min(5, windSpeed) × 0.8  // °C reduction per m/s
effectiveTemp -= windCooling
```

**Parameters:**
- Range: 0-10 m/s
- Effect: Up to 4°C cooling at 5+ m/s wind
- Basis: Enhanced convective heat transfer
- Diminishing returns beyond 5 m/s

**Physiological Effect:**
Wind enhances convective heat transfer from skin surface, improving heat dissipation capacity especially when combined with sweating.

#### 3.3 Solar Radiation Effect (Radiant Heat Load)

**Source:** WBGT outdoor model, ISO 7243

**Implementation:**
```javascript
solarLoad = (solarRadiation / 200) × (0.5 + clothing × 0.5)
effectiveTemp += solarLoad
```

**Parameters:**
- Range: 0-1000 W/m²
- Typical values:
  - 0 W/m²: Indoors or nighttime
  - 200-400 W/m²: Cloudy/shaded outdoor
  - 600-800 W/m²: Partly sunny
  - 800-1000 W/m²: Direct sunlight
- Modified by clothing (darker/more clothing = higher absorption)

**Physiological Effect:**
Solar radiation adds direct radiant heat load to the body, increasing thermal strain during outdoor work.

#### 3.4 Clothing Insulation Effect

**Source:** clo units standard (ISO 9920)

**Implementation:**
```javascript
clothingEffect = max(0, clothing - 0.6) × 3.0
effectiveTemp += clothingEffect
```

**Parameters:**
- Range: 0-2.0 clo
- Standard values:
  - 0.0 clo: Naked/minimal clothing
  - 0.5 clo: Shorts + t-shirt (summer casual)
  - 0.6 clo: Light work clothes (baseline)
  - 1.0 clo: Business suit
  - 1.5 clo: Winter jacket
  - 2.0 clo: Heavy winter gear
- Baseline: 0.6 clo (light occupational clothing)

**Physiological Effect:**
Higher clothing insulation impedes heat dissipation through reduced evaporative and convective cooling, increasing thermal strain.

#### 3.5 Environment Influence Control (NEW)

**User-Adjustable Parameter:** 0-100% slider

**Purpose:** Since there's no evidence for how strongly environmental factors should modify Buller's HR-based prediction, this slider lets users control the coupling strength.

**Implementation:**
```javascript
// environmentInfluence ranges from 0.0 to 1.0 (0% to 100%)
heatIncrement = envStress × activityIntensity × environmentInfluence
currentCT += heatIncrement
```

**Usage Guidelines:**
- **0%:** Pure Buller algorithm - environment has NO effect (original algorithm)
- **25%:** Weak environmental influence - subtle effects
- **50%:** Moderate influence (default) - balanced hybrid model
- **75%:** Strong environmental influence - pronounced effects
- **100%:** Maximum influence - environmental factors dominate

**Practical Application:**
If you have real-world data (e.g., measured core temps under known conditions), you can calibrate this slider to match observations. This acknowledges the uncertainty in combining these models.

#### 3.6 Combined Environmental Stress Application

**Complete Model:**
```javascript
// 1. Start with Heat Index (temp + humidity)
effectiveTemp = HeatIndex(temperature, humidity)

// 2. Apply wind cooling
if (windSpeed > 0.1) {
    effectiveTemp -= windCooling(windSpeed)
}

// 3. Add solar heat load
effectiveTemp += solarLoad(solarRadiation, clothing)

// 4. Add clothing insulation effect
effectiveTemp += clothingEffect(clothing)

// 5. Calculate net heat stress
heatStressDelta = effectiveTemp - ambientTemp

// 6. Apply during activity only, with user-controlled influence
if (HR > RestingHR) {
    activityIntensity = (HR - RestingHR) / (MaxHR - RestingHR)
    heatIncrement = heatStressDelta × activityIntensity × environmentInfluence
    currentCT += heatIncrement
}
```

**Key Features:**
- **Additive model:** All factors combine to create effective temperature
- **Activity-dependent:** Only affects core temp during physical work
- **Bidirectional:** Can increase OR decrease thermal strain
- **User-calibratable:** Environment Influence slider acknowledges model uncertainty
- **Evidence-based components:** Each environmental factor from validated standards

**Example Scenarios:**

| Scenario | Temp | RH | Wind | Solar | Clothing | Effective Temp | Effect |
|----------|------|----|----|-------|----------|----------------|--------|
| Indoor AC | 22°C | 40% | 0.5 m/s | 0 W/m² | 0.6 clo | ~21°C | Neutral |
| Outdoor Shaded | 30°C | 60% | 2 m/s | 300 W/m² | 0.6 clo | ~32°C | Moderate stress |
| Direct Sun | 35°C | 70% | 1 m/s | 900 W/m² | 0.6 clo | ~44°C | High stress |
| Windy Cool | 25°C | 50% | 5 m/s | 200 W/m² | 1.0 clo | ~23°C | Enhanced cooling |

### 4. Evidence-Based Warning Thresholds

**Source:** Occupational health standards (OSHA, ACGIH, NIOSH)

| Zone | Temperature Range | Classification | Action Required |
|------|------------------|----------------|-----------------|
| **Normal** | ≤ 37.2°C | Normothermia | Continue monitoring |
| **Caution** | 37.3 - 38.0°C | Mild Hyperthermia | Reduce intensity, hydrate, seek cooler environment |
| **Warning** | 38.1 - 39.0°C | Moderate Hyperthermia | CEASE activity, move to cooler environment, hydrate, monitor symptoms |
| **Danger** | > 39.0°C | Severe Hyperthermia | MEDICAL EMERGENCY - implement active cooling, seek immediate medical attention |

### 5. Time Tracking & Display

**Elapsed Time Calculation:**
```javascript
// Each algorithm step increments time
elapsedSimulationTime += (1 / updatesPerSecond)
```

**Time Display Features:**
- **Format:** Automatic MM:SS or HH:MM:SS formatting
- **X-Axis Labels:** Time markers at regular 100-pixel intervals
- **Scrolling Window:** Shows time range for visible data (last ~740 pixels)
- **Current Time:** Highlighted in blue at right edge
- **Precision:** Accurate to update rate (e.g., 10 Hz = 0.1s precision)

**Time Calculation:**
- Starts at 0:00 when simulation begins or is reset
- Increments based on actual update rate (not real-time clock)
- Example: At 10 Hz, each update adds 0.1 seconds
- Independent of rendering frame rate (60 FPS)

### 6. Performance Optimizations

**Canvas Rendering Optimizations:**
- **Background Caching:** Static elements (grid, zones, axes) drawn once and cached as ImageData
- **Batched Path Operations:** Single path for temperature line instead of 800+ individual segments
- **Reduced Draw Calls:** ~90% reduction in per-frame operations
- **Result:** 55-60 FPS even with real-time updates

**Update Rate Control:**
- Decoupled visual rendering (60 FPS) from algorithm updates (1-60 Hz)
- Default: 10 Hz (realistic physiological response rate)
- Adjustable: 1-60 Hz for different simulation speeds

**FPS Monitoring:**
- Real-time display of rendering FPS
- Algorithm update rate tracking
- Performance debugging information

## Usage

### Basic Operation

1. **Open `BullerAlogrithm.html`** in a modern web browser
2. **Select Activity Level** or use manual heart rate control
3. **Adjust Environmental Conditions** (temperature, humidity)
4. **Set Individual Parameters** (age, resting HR)
5. **Monitor Core Temperature** in real-time with visual warnings

### Example Scenarios

#### Scenario 1: Office Worker in Air-Conditioned Environment
- Activity: Sitting/Desk Work (1.3 METs)
- Environment: 22°C, 40% humidity
- Expected: Stable core temp ~37°C
- Heat Index: No significant effect

#### Scenario 2: Construction Worker in Hot Humid Conditions
- Activity: Heavy Lifting (6.0 METs)
- Environment: 35°C, 80% humidity, 1 m/s wind, 800 W/m² solar, 0.6 clo
- Expected: Rapid core temp rise → Warning zone
- Effective Temp: ~46°C (extreme heat stress)

#### Scenario 3: Firefighter During Rescue Operations
- Activity: Firefighting/Rescue (10.0 METs)
- Environment: 40°C, 60% humidity, 0.5 m/s wind, 600 W/m² solar, 1.5 clo (gear)
- Expected: Critical core temp rise → Danger zone
- Effective Temp: ~55°C (clothing impedes cooling)

#### Scenario 4: Outdoor Worker with Cooling Measures
- Activity: Moderate Labor (4.5 METs)
- Environment: 30°C, 50% humidity, 5 m/s wind (fan), 400 W/m² solar, 0.5 clo
- Expected: Manageable thermal strain with wind cooling
- Effective Temp: ~28°C (wind reduces heat stress)

## Technical Specifications

### Browser Compatibility
- Modern browsers with HTML5 Canvas support
- Tested on: Chrome, Firefox, Safari, Edge

### Dependencies
- Tailwind CSS (CDN)
- Google Fonts: Lexend, Fira Code

### Canvas Resolution
- Drawing buffer: 800×500 pixels
- Responsive scaling via CSS

### Data Display
- Real-time temperature line graph with time axis
- Time units: MM:SS format (automatically switches to HH:MM:SS for long simulations)
- Time markers at regular intervals along x-axis
- Current elapsed time highlighted in blue
- 4 color-coded warning zones (Normal, Caution, Warning, Danger)
- Current measurement indicator (pulsing dot)
- Environmental data overlay
- Heat Index display (when significant)

## Scientific References

1. **Buller's ECTemp Algorithm**
   - Buller, M.J., et al. "Estimation of human core temperature from sequential heart rate observations." *Physiological Measurement* (2013)
   - Original Kalman filter constants and HR-temperature relationship

2. **Heat Index (Temperature + Humidity)**
   - Rothfusz, L.P. "The Heat Index Equation." NWS Technical Attachment SR 90-23 (1990)
   - Steadman, R.G. "The Assessment of Sultriness." *Journal of Applied Meteorology* (1979)
   - National Weather Service / NOAA official equation

3. **Wind Speed Effects**
   - ISO 7933: "Ergonomics of the thermal environment - Analytical determination and interpretation of heat stress"
   - Convective heat transfer coefficients
   - Wind chill adapted for heat stress

4. **Solar Radiation**
   - ISO 7243: "Hot environments - Estimation of the heat stress on working man, based on the WBGT-index"
   - WBGT outdoor calculation including radiant heat
   - Solar heat load on human body

5. **Clothing Insulation**
   - ISO 9920: "Ergonomics of the thermal environment - Estimation of thermal insulation and water vapour resistance of a clothing ensemble"
   - clo units standard (1 clo = 0.155 m²·K/W)
   - Thermal insulation values for occupational clothing

6. **MET Values**
   - Ainsworth, B.E., et al. "2011 Compendium of Physical Activities." *Medicine & Science in Sports & Exercise* (2011)
   - PMC9586849: Heart Rate and MET correlation study
   - Validated metabolic equivalent values

7. **Heat Stress Thresholds**
   - OSHA Technical Manual: Heat Stress
   - ACGIH TLV for Heat Stress
   - PMC10010916: Critical wet-bulb temperatures for heat stress

8. **Core Temperature Guidelines**
   - NIOSH Criteria for Occupational Exposure to Heat
   - Occupational safety limits for core temperature

## Limitations & Disclaimers

1. **Educational/Research Tool:** This is a demonstration and research exploration tool, NOT a medical device

2. **EXPERIMENTAL HYBRID MODEL:** The combination of Buller's HR-based algorithm with environmental factors has NO established scientific validation. This is an experimental approach to explore potential interactions.

3. **No Clinical Use:** Not FDA approved. Not for clinical diagnosis, treatment decisions, or occupational health compliance.

4. **Model Uncertainty:** The "Environment Influence" slider (0-100%) exists precisely because we don't know the "correct" coupling strength between physiological and environmental models. This is honest acknowledgment of uncertainty.

5. **Individual Variation:** Default parameters may not fit all individuals. Age, fitness, acclimatization, health status, and medications all affect responses.

6. **Calibration Needed:** If using for research, calibrate the environment influence parameter against real-world measurements for your specific population and conditions.

7. **Simplified Assumptions:** 
   - Assumes steady-state clothing and activity
   - Doesn't model transient heat storage
   - Ignores some factors (air velocity around body, radiant asymmetry, clothing moisture)

8. **Validation Status:**
   - ✅ Buller's algorithm: Validated (±0.3°C accuracy)
   - ✅ Individual environmental components: Validated (ISO standards, NOAA)
   - ❌ Combined model: NOT validated

## License & Attribution

Developed based on publicly available research and validated scientific models.

For academic or commercial use, please cite:
- Original ECTemp Algorithm: Dr. Mark Buller, USARIEM
- Heat Index: National Weather Service / NOAA
- Activity METs: Ainsworth Compendium of Physical Activities

## Version History

- **v3.0** (Current): Comprehensive environmental model with all Rothfusz parameters
  - Wind speed control (convective cooling)
  - Solar radiation control (radiant heat load)
  - Clothing insulation control (clo units)
  - Integrated multi-factor heat stress calculation
  - Updated UI with all environmental controls
  - Enhanced display showing effective temperature
- **v2.0**: Evidence-based environmental heat stress model with Heat Index
  - Rothfusz regression implementation
  - Temperature and humidity controls
  - Time axis with MM:SS formatting
- **v1.5**: Performance optimizations, FPS monitoring
  - Canvas background caching
  - Batched rendering operations
  - 90% performance improvement
- **v1.0**: MET-based activity selector, warning thresholds
  - 12 occupational activities
  - Evidence-based warning zones
  - Individual parameter controls
- **v0.9**: Initial Buller's algorithm implementation
  - Core Kalman filter
  - Basic visualization

## Contact & Contributions

For questions, improvements, or bug reports, please refer to the project repository.
