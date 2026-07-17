# Measurement Algorithm

- Version: v0.5.2
- Updated: 2026-07-17

## Overview

Caster Gauge estimates the steering axis from gravity vectors measured by a smartphone fixed to an alignment gauge. The app treats normalized gravity vectors captured at multiple stationary steering positions as a point cloud in 3D phone coordinates, fits those points to a plane, and uses the plane normal as the estimated steering-axis direction.

The app then decomposes the estimated axis into vehicle vertical, lateral, and forward components to calculate absolute caster angle and an absolute SAI/KPI-like kingpin inclination reference value.

## Inputs

The primary sensor input is:

```js
DeviceMotionEvent.accelerationIncludingGravity
```

For each stationary state, the app samples:

```js
event.accelerationIncludingGravity.x
event.accelerationIncludingGravity.y
event.accelerationIncludingGravity.z
```

The samples are retained in a rolling window whose duration depends on the selected stillness profile. They are block-averaged, checked for raw vibration and long-term attitude stability, robustly averaged when a vibration profile is active, and normalized into one gravity unit vector. That normalized vector is stored as one capture.

The caster calculation does not use `DeviceOrientation` heading values, integrated gyro angles, steering angle, or camber angle as independent inputs.

## Calibration

The phone is assumed to be mounted upright with the screen facing outward from the wheel. The current implementation treats the phone's screen-out direction as the vehicle-outward direction.

The straight-ahead baseline records the gravity vector with the steering in the straight-ahead position. This baseline is used to construct vehicle reference directions in phone coordinates:

- vertical up
- vehicle lateral direction
- vehicle forward direction

The straight-ahead baseline vector is not included in the current implementation's plane-fit measurement point set. Plane fitting uses only the measurement points stored in `state.points`.

The baseline button arms baseline capture instead of sampling immediately. Once the recent sensor window passes the stillness test, the app records the straight-ahead baseline automatically. Immediately after that successful baseline capture, the app attempts to add the first capture from the same still window. A still session then accepts at most two automatic captures. Those captures share one session ID and are combined into one representative vector for fitting.

## Measurement Points

The wheel is moved through multiple steering positions. At each position, the vehicle and phone must be stationary before a gravity vector is accepted.

Captures can be added manually or automatically. Automatic addition uses:

- stillness over a sampling window
- a cooldown time after the last automatic point
- a maximum of two automatic captures per still session

When the phone first becomes still, the current implementation adds the first capture immediately and can add one more capture after the cooldown interval if the phone remains still. It then stops for that still session. The still-session counter resets only after the phone leaves the stillness condition and later becomes still again. The two captures are averaged into one normalized session representative, so they improve the representative at that position without being counted as two independent steering positions.

Version 0.5.2 provides three stillness profiles:

| Profile | Stillness/averaging window | Min samples | Raw axis std | Raw magnitude std | Block | Block angle RMS | Direction drift | Block magnitude std | Trim | Cooldown |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| standard | 1200 ms | 20 | 0.15 | 0.20 | 300 ms | 0.20 deg | 0.35 deg | 0.08 | 0% | 1000 ms |
| vibration low | 3000 ms | 45 | 0.80 | 1.20 | 500 ms | 0.45 deg | 0.55 deg | 0.25 | 10% | 1500 ms |
| vibration high | 5000 ms | 75 | 2.00 | 3.00 | 1000 ms | 1.00 deg | 0.80 deg | 0.50 | 20% | 2500 ms |

All profiles require at least 85% of the configured time span, automatic capture requires a passing stability result, and each still session accepts at most two captures. The app retains up to 40 raw captures, uses up to 32 sessions for jackknife uncertainty, and renders the latest 40 rows plus the baseline.

The 1.2, 3, and 5 second values are rolling sensor-data window lengths. Each window is used both to decide whether the attitude is stable and to calculate the averaged gravity vector stored for that capture. They are not fixed delays added after a separate stillness decision. The second capture at one position uses the same rolling-window process after the profile-specific cooldown has elapsed.

Crossing a stillness threshold once does not rearm automatic capture. The failed condition must continue for 500, 750, or 1000 ms for the standard, vibration-low, or vibration-high profile, and the filtered gravity direction must also move at least 0.20, 0.30, or 0.40 degrees from the last captured direction. The angle is checked again when a stable result returns. Until these conditions pass, the app keeps the current session locked and displays a held state. This hysteresis prevents threshold flicker at one physical steering position from creating multiple independent sessions.

### Vibration filtering

The vibration profiles do not rely only on relaxed raw standard-deviation thresholds:

1. Retain 3 or 5 seconds of rolling acceleration-including-gravity samples.
2. Divide the window into 500 or 1000 ms blocks and average each block independently.
3. Compute a component-wise trimmed mean of the block vectors, then normalize it as the candidate gravity direction.
4. Check raw per-axis and magnitude standard deviations as upper safety limits.
5. Compute RMS angular deviation of the block directions from the candidate direction.
6. Compare the mean direction of the first third of blocks with the last third to detect slow steering movement.
7. Check the standard deviation of block-mean magnitudes.
8. Accept the capture only when every applicable check passes.

Periodic vibration with approximately zero mean is reduced by the block and multi-second averages. Slow attitude movement remains visible in the block-angle and first/last-direction checks. The filter cannot remove a vibration-induced sensor bias or mechanical movement whose time average changes with steering position.

When the retained-capture limit is exceeded, the app first prefers captures from sessions that still contain another capture. The trim score then combines robust-plane residual, angular-density score, and age, so outliers, redundant captures, and over-represented dense regions are removed before sparse useful positions when possible.

The steering angle in degrees is not acquired, calculated, or used as an input.

## Plane Fit

The app treats the normalized gravity vectors as points in 3D space.

1. Group raw captures by stationary-session ID.
2. Average and normalize the captures in each session to make one representative vector.
3. Compute angular-density weights for the session representatives. The normalized density weight is capped to the range 0.5 through 2.0.
4. Compute a weighted point-cloud mean and weighted 3x3 covariance matrix.
5. Perform eigendecomposition of the symmetric covariance matrix and select the eigenvector corresponding to the smallest eigenvalue.
6. Select a robust initial plane from the full fit and candidates that each omit one session.
7. Compute signed plane residuals and update the weights with Tukey's biweight function. A clear outlier can receive zero weight. If fewer than five nonzero positions would remain, fall back to Huber weights.
8. Repeat the weighted fit until the weights converge or 10 iterations have completed.

That eigenvector is the normal of the robust, density-weighted least-squares plane fitted to the measured session representatives. The app uses this plane normal as the estimated steering-axis direction. The robust initialization and Tukey stage reject a clear outlier before the 40-capture retention limit is reached; they do not silently turn the outlier into a high-confidence sparse point.

This implementation does not construct a virtual 3D coordinate point on the wheel axis for the straight-ahead position or for two steered positions. It also does not compute a plane through exactly three virtual points. It fits a plane statistically to five or more stationary-session representative gravity vectors.

## Caster And SAI/KPI-Like Values

From the fixed screen-out mounting assumption and straight-ahead baseline, the app constructs vehicle reference directions in phone coordinates:

- `verticalUp`
- `lateral`
- `forward`

The estimated steering axis is projected onto those directions.

Caster angle is calculated from the steering-axis vertical and forward components, then displayed as an absolute value:

```js
casterRad = Math.atan2(
  dot(axis, forward),
  dot(axis, verticalUp)
);
```

The SAI/KPI-like kingpin inclination reference value is calculated from the steering-axis vertical and lateral components, then displayed as an absolute value:

```js
saiRad = Math.atan2(
  dot(axis, lateral),
  dot(axis, verticalUp)
);
```

Signed caster and kingpin-angle-equivalent values are kept in CSV for diagnostics, but the main UI displays absolute values to avoid left/right wheel sign handling in normal use.

## Fit Quality And Uncertainty

The app displays robust weighted RMS residual, quality text, and caster uncertainty. Quality assessment uses:

- independent stationary-session count
- maximum pairwise angular span of the session gravity vectors
- separation ratio between the middle and smallest covariance eigenvalues
- robust outlier count
- jackknife standard error

The previous maximum-eigenvalue / middle-eigenvalue test was removed. That ratio becomes large for a narrow arc and therefore cannot establish that the steering range is sufficient. Version 0.5.0 checks angular span directly and uses the middle/smallest eigenvalue separation to assess whether the plane normal is distinguishable from residual noise.

Caster uncertainty is estimated with a session-level delete-one jackknife:

1. Remove one complete stationary session, including both captures when present.
2. Recalculate the robust weighted plane fit and caster angle.
3. Repeat for every session.
4. Let the resulting estimates be `theta_i` and their mean be `theta_bar`.
5. Calculate the jackknife standard error as shown below.

```text
SE = sqrt((n - 1) / n * sum((theta_i - theta_bar)^2))
95% width = 1.96 * SE
```

The displayed `SE` is this standard error in degrees. Unlike the previous implementation, repeated captures at one stationary position are not treated as independent points, and the jackknife scaling factor is included. The displayed `±` value is the approximate 95% internal variation width. It remains an empirical fit-stability indicator, not a formal metrology certificate, and does not include wheel-to-phone mounting direction, toe/straight-ahead reference error, suspension compliance, or accelerometer calibration bias.

## Not Used

The current implementation does not use:

- steering angle as an independent input
- camber angle as an independent input or intermediate parameter
- `DeviceOrientation` heading values for caster calculation
- integrated gyro angle for caster calculation
- virtual 3D points constructed on the wheel axis
- a plane defined only by three virtual coordinate points

Instead, the steering axis is estimated directly by session-aggregated, robust density-weighted least-squares plane fitting of multiple measured gravity vectors.
