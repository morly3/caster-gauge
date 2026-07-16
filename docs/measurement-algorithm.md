# Measurement Algorithm

- Version: v0.4.9
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

The baseline button arms baseline capture instead of sampling immediately. Once the recent sensor window passes the stillness test, the app records the straight-ahead baseline automatically. Immediately after that successful baseline capture, the app attempts to add the first measurement point from the same still window. A still session then accepts at most two automatic measurement points.

## Measurement Points

The wheel is moved through multiple steering positions. At each position, the vehicle and phone must be stationary before a gravity vector is accepted.

Measurement points can be added manually or automatically. Automatic addition uses:

- stillness over a sampling window
- a cooldown time after the last automatic point
- a maximum of two automatic points per still session

When the phone first becomes still, the current implementation adds the first point immediately and can add one more point after the cooldown interval if the phone remains still. It then stops for that still session. The still-session counter resets only after the phone leaves the stillness condition and later becomes still again. Duplicate or dense captures are also handled by density weighting during fitting.

Current v0.4.9 capture thresholds are conservative after field testing showed that permissive high-rate capture did not improve the displayed uncertainty enough and could overload mobile browsers:

- sample window: 1200 ms
- minimum samples: 20
- maximum per-axis standard deviation: 0.08
- maximum magnitude standard deviation: 0.12
- automatic-add cooldown: 1000 ms
- automatic add requires stillness: yes
- automatic points per still session: 2
- retained measurement points: 40
- points used for leave-one-out uncertainty: up to 32
- rendered point rows: latest 40 plus baseline

When the retained measurement-point limit is exceeded, the app preferentially removes automatic points with a high trim score. The score combines plane residual, angular-density score, and age, so outliers and over-represented dense regions are removed before sparse useful regions when possible.

The steering angle in degrees is not acquired, calculated, or used as an input.

## Plane Fit

The app treats the normalized gravity vectors as points in 3D space.

1. Compute the point-cloud mean.
2. Subtract the mean from each point.
3. Compute angular-density weights. Points with many nearby neighbors receive lower relative weight, while points in sparse steering regions receive higher relative weight.
4. Accumulate a weighted 3x3 covariance matrix from those centered vectors.
5. Perform eigendecomposition of the symmetric covariance matrix.
6. Select the eigenvector corresponding to the smallest eigenvalue.

That eigenvector is the normal of the weighted least-squares plane fitted to the measured gravity-vector points. The app uses this plane normal as the estimated steering-axis direction.

This implementation does not construct a virtual 3D coordinate point on the wheel axis for the straight-ahead position or for two steered positions. It also does not compute a plane through exactly three virtual points. It fits a plane statistically to five or more measured gravity vectors.

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

The app displays weighted RMS residual, quality text, and caster uncertainty. Caster uncertainty is estimated by leave-one-out recalculation:

1. Remove one measurement point.
2. Recalculate the weighted plane fit and caster angle.
3. Repeat for every measurement point.
4. Compute the standard deviation of those caster angles.

The displayed `σ` is that standard deviation in degrees. The displayed `±` value is approximately `1.96 * σ`, used as a rough 95% variation range. This value is an empirical fit-stability indicator, not a formal metrology certificate.

## Not Used

The current implementation does not use:

- steering angle as an independent input
- camber angle as an independent input or intermediate parameter
- `DeviceOrientation` heading values for caster calculation
- integrated gyro angle for caster calculation
- virtual 3D points constructed on the wheel axis
- a plane defined only by three virtual coordinate points

Instead, the steering axis is estimated directly by density-weighted least-squares plane fitting of multiple measured gravity vectors.
