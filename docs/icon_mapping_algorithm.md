# Weather Icon Mapping Fix

## Problem

The issue #6 reported that forecast icons were not displaying in Home Assistant. Investigation revealed that the WeatherXM API returns icon values that were not present in the `ICON_TO_CONDITION_MAP` dictionary, causing the integration to return `"unknown"` which Home Assistant cannot render.

## Investigation Results

### API Icons Found

By querying the WeatherXM API directly, we discovered the following icon values are currently returned:

**Daily Forecast Icons:**
- extreme-day-drizzle
- extreme-day-rain
- fog
- overcast-day
- partly-cloudy-day
- partly-cloudy-day-drizzle

**Hourly Forecast Icons:**
- clear-day
- clear-night
- fog
- haze-night
- overcast
- overcast-drizzle
- partly-cloudy-day
- partly-cloudy-night
- partly-cloudy-night-drizzle
- thunderstorms-light-rain

### Missing Mappings (8 total)

The following 8 icons were **missing** from the original mapping:

1. `extreme-day-drizzle` → mapped to `snowy-rainy` (extreme drizzle conditions)
2. `extreme-day-rain` → mapped to `rainy` (heavy rain)
3. `haze-night` → mapped to `fog` (nighttime haze)
4. `overcast-day` → mapped to `cloudy` (fully overcast)
5. `overcast-drizzle` → mapped to `rainy` (light rain/drizzle under clouds)
6. `partly-cloudy-day-drizzle` → mapped to `rainy` (light drizzle with some clouds)
7. `partly-cloudy-night-drizzle` → mapped to `rainy` (nighttime drizzle with clouds)
8. `thunderstorms-light-rain` → mapped to `lightning-rainy` (thunderstorm with light rain)

## Solution

### Mapping Algorithm

The icon mapping follows these principles:

1. **Safety first**: When uncertain, map to more common conditions
2. **Semantic correctness**: Match the weather phenomenon accurately
3. **HA compatibility**: Only use valid Home Assistant weather conditions

### Mapping Strategy by Icon Type

#### Extreme Weather Icons
- `extreme-day-drizzle` → `snowy-rainy` (mixed precipitation)
- `extreme-day-rain` → `rainy` (heavy rain event)

#### Haze/Fog Variants
- `haze-night` → `fog` (visibility reduction, night variant)

#### Overcast Variants
- `overcast-day` → `cloudy` (complete cloud cover)
- `overcast-drizzle` → `rainy` (light precipitation)

#### Partly Cloudy with Precipitation
- `partly-cloudy-day-drizzle` → `rainy` (scattered clouds with drizzle)
- `partly-cloudy-night-drizzle` → `rainy` (night variant)

#### Thunderstorm Variants
- `thunderstorms-light-rain` → `lightning-rainy` (thunder with light rain)

## Implementation

### Code Changes

Updated `ICON_TO_CONDITION_MAP` in `weather.py` to include all 8 missing icons:

```python
ICON_TO_CONDITION_MAP = {
    # ... existing mappings ...
    "extreme-day-drizzle": "snowy-rainy",  # NEW
    "extreme-day-rain": "rainy",            # NEW
    "haze-night": "fog",                    # NEW
    "overcast-day": "cloudy",               # NEW
    "overcast-drizzle": "rainy",            # NEW
    "partly-cloudy-day-drizzle": "rainy",   # NEW
    "partly-cloudy-night-drizzle": "rainy", # NEW
    "thunderstorms-light-rain": "lightning-rainy", # NEW
}
```

### Fallback Behavior

If an unknown icon is encountered, the code still uses:
```python
ICON_TO_CONDITION_MAP.get(icon, "unknown")
```

This ensures backward compatibility and graceful degradation.

## Testing

### API Validation
- Queried live WeatherXM API endpoints
- Extracted all unique icon values from forecast data
- Verified 100% coverage of current API icons

### Home Assistant Validation
- All mapped conditions are valid HA weather conditions
- Icons now display correctly in weather cards
- Both daily and hourly forecasts render properly

## Coverage Statistics

**Before Fix:**
- Total API icons: 14
- Mapped icons: 6
- Coverage: 43%
- Missing icons: 8 (57%)

**After Fix:**
- Total API icons: 14
- Mapped icons: 14
- Coverage: 100%
- Missing icons: 0

## Future Maintenance

### Adding New Icons

When WeatherXM adds new icon values:

1. **Identify the icon value** from API responses (check logs or query API)
2. **Determine appropriate HA condition** using semantic mapping
3. **Add to ICON_TO_CONDITION_MAP** in alphabetical order
4. **Test with real data** to ensure correct display
5. **Update this documentation**

### Monitoring for API Changes

To detect new icons:
```python
# Add to __init__.py in async_update_data()
for device in devices:
    for daily in device.get('forecast', []):
        icon = daily.get('daily', {}).get('icon')
        if icon and icon not in ICON_TO_CONDITION_MAP:
            _LOGGER.warning("Unknown daily icon: %s", icon)
    
    for hourly in daily.get('hourly', []):
        icon = hourly.get('icon')
        if icon and icon not in ICON_TO_CONDITION_MAP:
            _LOGGER.warning("Unknown hourly icon: %s", icon)
```

## Related Issues

- Fixes #6 - Forecast icons lost
- Related to API changes in WeatherXM v3.42.17

## References

- [Home Assistant Weather Conditions](https://www.home-assistant.io/integrations/weather/#condition-mapping)
- [WeatherXM API Documentation](https://api.weatherxm.com/api/v1/docs)
