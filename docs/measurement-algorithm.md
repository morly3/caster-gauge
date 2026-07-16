# Measurement Algorithm

- Version: v5.0.0
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

The samples are averaged over a fixed duration, checked for stillness, and normalized into one gravity unit vector. That normalized vector is stored as one measurement point.

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

Current v5.0.0 capture thresholds are conservative after field testing showed that permissive high-rate capture did not improve the displayed uncertainty enough and could overload mobile browsers:

- sample window: 1200 ms
- minimum samples: 20
- maximum per-axis standard deviation: 0.08
- maximum magnitude standard deviation: 0.12
- automatic-add cooldown: 1000 ms
- automatic add requires stillness: yes
- automatic captures per still session: 2
- retained raw captures: 40
- sessions used for jackknife uncertainty: up to 32
- rendered point rows: latest 40 plus baseline

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

The previous maximum-eigenvalue / middle-eigenvalue test was removed. That ratio becomes large for a narrow arc and therefore cannot establish that the steering range is sufficient. Version 5.0.0 checks angular span directly and uses the middle/smallest eigenvalue separation to assess whether the plane normal is distinguishable from residual noise.

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
