# LRTP Data Dictionary

## Primary VehiclePositions Table
### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Trip Service Date | The service date of the trip departure (service date transitions at 4AM) | Date | `date(DATEPARSE("yyyyMMdd", [vehicle.trip.start_date]))`
Latest Data Date | The day that the data has most recently been updated | Date | `{MAX([Trip Service Date])}`
2 Weeks Ago from Latest Data Date | Two weeks before the day that the data has most recently been updated | Date | `{MAX([Trip Service Date])-15}`
VehiclePositions Unique Daily Trip Identifier | Field to uniquely identify trip departures based on Trip Service Date and Trip ID | String | `str([Trip Service Date]) + " " + [vehicle.trip.trip_id]`
Departure Time | The time that the trip departed (the first time that the vehicle starts moving towards the following station) | Date & Time | `{ FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN([VehiclePositions feed_timestamp] END)}`
Departure Stop ID | The stop ID of the station that the trip is departing to | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.stop_id] END )}`
Route | Trip departure route | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.trip.route_id] END )}`
Vehicle Consist | Vehicle assignment for the trip departure | String | `{fixed [VehiclePositions Unique Daily Trip Identifier]: min(if [VehiclePositions feed_timestamp] = [Departure Time] then [vehicle.vehicle.label] END )}`
Trip Terminal | The terminal station that the trip departed from | String | `CASE [Departure Stop ID] WHEN "70502" THEN "Union Square" WHEN "70510" THEN "Medford/Tufts" WHEN "70110" THEN "Boston College" WHEN "70236" THEN "Cleveland Circle" WHEN "70162" THEN "Riverside" WHEN "70274" THEN "Mattapan" END`
Day of Week | Day of the week of the trip service date | String | `DATENAME('weekday', [Trip Service Date])`
Hour | Hour of the trip departure time | String | `IF (DATEPART('hour',[Departure Time]))=0 THEN '12AM' ELSEIF (DATEPART('hour',[Departure Time]))=12 THEN '12PM' ELSEIF (DATEPART('hour',[Departure Time]))>12 THEN STR((DATEPART('hour',[Departure Time]))-12) + 'PM' ELSE STR((DATEPART('hour',[Departure Time]))) + 'AM' END`
Trip Departure Rank per Terminal | Numerical order of the trip departure per terminal | Number (whole) | `{PARTITION [VehiclePositions Unique Daily Trip Identifier]: {ORDERBY [Departure Time] ASC,[vehicle.trip.trip_id] ASC: RANK_DENSE() }}` |

### Data Filters
- `vehicle.trip.revenue`=TRUE
- `vehicle.trip.trip_id`!=NULL
- `vehicle.stop_id`!=NULL
- `feed_timestamp`!=NULL
- `Departure Time`!=NULL

---

## Secondary VehiclePositions Table
### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Next Trip Departure Rank per Terminal | Numerical order of the next trip departure per terminal | Number (whole) | `[Trip Departure Rank per Terminal]-1`
Next Vehicle Consist | Vehicle assignment for the next trip departure | String | Rename `[Vehicle Consist]`
Next Trip Terminal | The terminal station that the next trip departed from | String | Rename `[Trip Terminal]`
Next Trip Service Date | The service date of the next trip departure | Date | Rename `[Trip Service Date]`
Next Departure Time | The time that the next trip departed | Date & Time | Rename `[Departure Time]` |
- Left join the primary VehiclePositions table and Secondary VehiclePositions tables on `Trip Departure Rank per Terminal`=`Next Trip Departure Rank per Terminal`, `Trip Service Date`=`Next Trip Service Date`, and `Trip Terminal`=`Next Trip Terminal`

---

## Tertiary VehiclePositions Table
### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Trip Departure Rank per Terminal After Next | Numerical order of the trip departure after next per terminal | Number (whole) | `[Trip Departure Rank per Terminal]-2`
Vehicle Consist After Next | Vehicle assignment for the trip departure after next | String | Rename `[Vehicle Consist]`
Trip Terminal After Next | The terminal station that the trip after next departed from | String | Rename `[Trip Terminal]`
Trip Service Date After Next | The service date of the trip departure after next | Date | Rename `[Trip Service Date]`
- Left join the primary VehiclePositions table and Tertiary VehiclePositions tables on `Trip Departure Rank per Terminal`=`Trip Departure Rank per Terminal After Next`, `Trip Service Date`=`Trip Service Date After Next`, and `Trip Terminal`=`Trip Terminal After Next`

---

## Quaternary VehiclePositions Table
### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Prior Trip Departure Rank per Terminal | Numerical order of the prior trip departure per terminal | Number (whole) | `[Trip Departure Rank per Terminal]+1`
Prior Vehicle Consist | Vehicle assignment for the prior trip departure | String | Rename `[Vehicle Consist]`
Prior Trip Terminal | The terminal station that the prior trip departed from | String | Rename `[Trip Terminal]`
Prior Trip Service Date | The service date of the prior trip departure | Date | Rename `[Trip Service Date]`
-	Left join the primary VehiclePositions table and Quaternary VehiclePositions tables on `Trip Departure Rank per Terminal`=`Prior Trip Departure Rank per Terminal`, `Trip Service Date`=`Prior Trip Service Date`, and `Trip Terminal`=`Prior Trip Terminal`

---

## Union of all VehiclePositions tables
- Perform a data union on all of the VehiclePositions tables

### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Number of Cars | The total number of cars in a trip departure's vehicle consist | Number (whole) |  `{ FIXED [VehiclePositions Unique Daily Trip Identifier]: MAX(IF(LEN([Vehicle Consist])=4) THEN 1 ELSEIF (LEN([Vehicle Consist])=9) THEN 2 END)}` |
Vehicle Consist Backwards | If there are 2 cars in a trip's vehicle consist then swap their order | String | `if([Number of Cars]=2) then (RIGHT(STR([Vehicle Consist]), 4) + "-" + LEFT(STR([Vehicle Consist]), 4)) else [Vehicle Consist] end`
Gap to Next Terminal Departure | Gap in minutes between a trip departure and the following trip departure | Number (whole) | `{ FIXED [VehiclePositions Unique Daily Trip Identifier]: MAX(DATEDIFF('minute',[Departure Time],[Next Departure Time]))}`
False Positive Departure? | Identify whether a trip is a false positive departure, likely as the result of a bad AVI read | String | `IF([Number of Cars]=2 AND ([Vehicle Consist]=[Prior Vehicle Consist] OR [Vehicle Consist Backwards]=[Prior Vehicle Consist]) AND ([Vehicle Consist]!=[Next Vehicle Consist] OR [Vehicle Consist Backwards]!=[Next Vehicle Consist])) THEN "False Positive" ELSEIF([Number of Cars]=2 AND ([Vehicle Consist]!=[Prior Vehicle Consist] OR [Vehicle Consist Backwards]!=[Prior Vehicle Consist]) AND ([Vehicle Consist]=[Next Vehicle Consist] OR [Vehicle Consist Backwards]=[Next Vehicle Consist])) THEN "" ELSEIF ([Number of Cars]=1 AND ([Gap to Next Terminal Departure]<5 OR ISNULL([Gap to Next Terminal Departure]))) THEN  (IF (CONTAINS([Next Vehicle Consist],[Vehicle Consist]) OR CONTAINS([Vehicle Consist After Next],[Vehicle Consist]))  THEN "False Positive"  ELSE "" END) ELSE "" END`

---

## Primary TripUpdates Table
### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Prediction Service Date | Format `trip_update.trip.start_date` as a Date | Date | `date(DATEPARSE("yyyyMMdd", [trip_update.trip.start_date]))`
TripUpdate Unique Daily Trip Identifier | Field to uniquely identify the trip departure based on trip service date and trip ID | String | `str([Prediction Service Date]) + " " + [trip_update.trip.trip_id]`
TripUpdate Unique Prediction ID | Value to uniquely identify the prediction based on the generated prediction time, the predicted departure time, and trip ID | String | `str([TripUpdate feed_timestamp]) + " " + str([trip_update.stop_time_update.departure.time]) + " " + [trip_update.trip.trip_id]`
Prediction Rank per Departure | Numerical order of the prediction per trip departure | Number (whole) | `{PARTITION [TripUpdate Unique Daily Trip Identifier]: {ORDERBY [TripUpdates feed_timestamp] ASC: RANK_DENSE() }}`

### Data Filters
- `trip_update.trip.revenue`=TRUE
- `trip_update.trip.schedule_relationship`!=CANCELED
- `trip_update.stop_time_update.schedule_relationship`!=SKIPPED
- `trip_update.stop_time_update.departure.time`!=NULL
- `Predictions After Departure Time`=FALSE

---

## Secondary TripUpdates Table
### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Next Prediction Rank per Departure | Numerical order of the next prediction per trip departure | Number (whole) | `[Prediction Rank per Departure]-1`
Next Prediction Generated Time | The time that the next trip prediction was generated | Date & Time | Rename `[TripUpdate feed_timestamp]`
Next TripUpdate Unique Daily Trip Identifier | Value to uniquely identify the next prediction based on the generated prediction time, the predicted departure time, and trip ID | String | Rename `[TripUpdate Unique Daily Trip Identifier]`
- Full outer join the primary joined data table and secondary joined data tables on `Prediction Rank per Departure`=`Next Prediction Rank per Departure` and `TripUpdate Unique Daily Trip Identifier`=`Next TripUpdate Unique Daily Trip Identifier`

---

## Joined Data Table
- Left join the VehiclePositions and TripUpdates tables on `VehiclePositions Unique Daily Trip Identifier`=`TripUpdate Unique Daily Trip Identifier`
### Custom Calculations
Field Name | Description | Type | Query |
--- | --- | --- |  --- |
Prediction Generated after Terminal Departure | Identify whether a prediction was generated after the trip departure time | Boolean |  `IF (DATEDIFF('second',[TripUpdates feed_timestamp],[Departure Time]) <= 0) THEN TRUE ELSE FALSE END` |
Advance Notice (minutes) | Amount of time in minutes that a prediction was generated prior to the trip departure time | Number (whole) | `DATEDIFF('minute',[TripUpdates feed_timestamp],[Departure Time])`
Advance Notice (minutes) per Departure | Amount of time in minutes that the first prediction of a trip was generated prior to the trip departure time | Number (whole) | `ZN({ FIXED [VehiclePositions Unique Daily Trip Identifier]: MAX([Advance Notice (minutes)]) })`
Time that Departure was First Predicted | The first time that a prediction was generated for a trip departure | String | `IF (ISNULL({ FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN([TripUpdates feed_timestamp])})) THEN "No prediction was made" ELSE STR({ FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN([TripUpdates feed_timestamp])}) END`
Kind | Identify trip kind based on the departure uncertainity | String | `IF ([trip_update.stop_time_update.departure.uncertainty]='60') then "Mid-trip" elseif ([trip_update.stop_time_update.departure.uncertainty]='120') then "At Terminal" elseif ([trip_update.stop_time_update.departure.uncertainty]='360') then "Reverse Trip" elseif (ISNULL([trip_update.stop_time_update.departure.uncertainty]) and ISNULL([Earliest Prediction Generated Time per Trip])) then "No Predictions" else str([trip_update.stop_time_update.departure.uncertainty]) end`
Bin | Categorization for the amount of advance notice of the prediction in minutes | String | `IF([Advance Notice (minutes)]>=0 AND [Advance Notice (minutes)]<3) then "0-3 min" ELSEIF ([Advance Notice (minutes)]>=3 AND [Advance Notice (minutes)]<6) then "3-6 min" ELSEIF ([Advance Notice (minutes)]>=6 AND [Advance Notice (minutes)]<12) then "6-12 min" ELSEIF ([Advance Notice (minutes)]>=12 AND [Advance Notice (minutes)]<=30) then "12-30 min" elseif ([Advance Notice (minutes)]>30) then "30+ min" elseif ISNULL([Advance Notice (minutes)]) then "No Predictions" end`
Actual - Predicted | The amount of time in seconds between the predicted departure time and the actual departure time | Number (whole) | `DATEDIFF('second',[Departure Time],[trip_update.stop_time_update.departure.time])`
Mean Error | Mean error of the amount of time in seconds between the predicted departure time and the actual departure time | Number (decimal) | `AVG([Actual - Predicted])`
Root Mean Squared Error | Root mean squared error of the amount of time in seconds between the predicted departure time and the actual departure time | Number (decimal) | `SQRT(AVG(SQUARE([Actual - Predicted])))`
Is Accurate? | Identify whether a prediction is accurate based on the IBI/Denver Methodology | String | `IF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="0-3 min") THEN (IF([Actual - Predicted]>=-60 AND [Actual - Predicted]<=60) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="3-6 min") THEN (IF([Actual - Predicted]>=-90 AND [Actual - Predicted]<=120) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="6-12 min") THEN (IF([Actual - Predicted]>=-150 AND [Actual - Predicted]<=210) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="12-30 min") THEN (IF([Actual - Predicted]>=-240 AND [Actual - Predicted]<=360) THEN "Accurate" ELSE "Inaccurate" END) ELSEIF([TripUpdate Unique Daily Trip Identifier]=[VehiclePositions Unique Daily Trip Identifier] and [Bin]="30+ min") THEN (IF([Actual - Predicted]>=-240 AND [Actual - Predicted]<=360) THEN "Accurate" ELSE "Inaccurate" END) end`
Prediction Available for Departure? | Identify whether a trip departure has any associated predictions | Boolean | `IF(ISNULL([TripUpdate Unique Daily Trip Identifier]) AND NOT ISNULL([VehiclePositions Unique Daily Trip Identifier])) THEN FALSE ELSE TRUE END`
\# Predictions | The total number of terminal predictions generated whose predicted terminal departure time matched to its corresponding actual departure time based on the trip service date and trip ID | Number (whole) | `COUNTD([TripUpdate Unique Prediction ID])`
\# Accurate Predictions | The number of terminal predictions generated whose predicted terminal departure time matched to its corresponding actual departure time based on the trip service date and trip ID, and passes the accuracy standard based on the IBI/Denver Methodology | Number (whole) | `ZN((COUNTD(IF([Is Accurate?]="Accurate") THEN ([TripUpdate Unique Prediction ID]) end)))`
% Accuracy | Percentage of accurate predictions out of the total number predictions generated | Whole (decimal) | `[# Accurate Predictions]/[# Predictions]`
\# Departures | The total number of terminal departures | Number (whole) | `IFNULL(COUNTD([VehiclePositions Unique Daily Trip Identifier]), 0)`
\# Predicted Departures | The total number of terminal departures that had a predicted terminal departure time matching to its corresponding actual departure time based on trip service date and trip ID. | Number (whole) | `IFNULL(countd(IF([Prediction Available for Departure?]=TRUE) THEN ([VehiclePositions Unique Daily Trip Identifier]) end),0)`
\# Departures with Continuous Coverage | The total number of terminal departures with continuous prediction coverage | Number (whole) | `ZN((COUNTD(IF([Continuous?]=TRUE) THEN ([VehiclePositions Unique Daily Trip Identifier]) end)))`
% Departures with Continuous Coverage | Percentage of departures with continuous prediction coverage out of the total number of departures | Number (decimal) | `[# Departures with Continuous Coverage]/[# Departures]`
Next Prediction Generated Time or Departure Time | For the last prediction generated for a departure get the trip departure time, otherwise get the next time that a prediction was generated for the departure | Date & Time | `IF(ISNULL([Next Prediction Generated Time])) THEN [Departure Time] else [Next Prediction Generated Time] end`
Gap to Next Prediction | The amount of time in seconds between the time that the prediction was generated and the next time that a prediction was generated for a trip departure. If its the last prediction generated for a departure, get the amount of time in seconds between the time that the prediction was generated and the trip departure time | Number (whole) | `zn(DATEDIFF('second', [TripUpdates feed_timestamp],[Next Prediction Generated Time or Departure Time]))`
Continuous Prediction with Min. Advance Notice | Determine which departures have continuous prediction coverage for predictions | Number (whole) | `IF([Gap to Next Prediction]>20 and [Advance Notice (minutes)]<=[Min. Advance Notice (minutes)]) THEN 1 ELSEIF([Gap to Next Prediction]<=20 and [Advance Notice (minutes)]<=[Min. Advance Notice (minutes)]) THEN 0 ELSEIF ([Advance Notice (minutes)]>[Min. Advance Notice (minutes)]) THEN NULL END`
Prediction Gaps with Min. Advance Notice? | The total number of prediction gaps that occurred | Boolean | `(IF(({ FIXED [VehiclePositions Unique Daily Trip Identifier]: SUM([Continuous Prediction with Min. Advance Notice])})=0) THEN FALSE ELSE TRUE END)`
Departure Advance Notice >= Min. Advance Notice | ... | Boolean | `IF([Advance Notice (minutes) per Departure]>=[Min. Advance Notice (minutes)]) THEN TRUE ELSE FALSE END`
Departure Advance Notice <= Min. Advance Notice Min Time | ... | Boolean | `{FIXED [VehiclePositions Unique Daily Trip Identifier]: MIN(IF [Advance Notice (minutes)]<=[Min. Advance Notice (minutes)] THEN [Prediction Generated Time] END)}`
Departure Advance Notice > Min. Advance Notice Max Time | ... | Boolean | `{FIXED [VehiclePositions Unique Daily Trip Identifier]:  MAX(IF [Advance Notice (minutes)]>[Min. Advance Notice (minutes)] THEN [Prediction Generated Time] END)}`
Continuous Coverage Between Advance Notice > Min. Advance Notice and Advance Notice <= Min. Advance Notice Predictions | ... | Boolean | `IF ((DATEDIFF('second',[Departure Advance Notice > Min. Advance Notice Max Time],[Departure Advance Notice <= Min. Advance Notice Min Time]))>20) THEN FALSE ELSE TRUE END`
Continuous? | Identify whether a trip departure has continuous prediction coverage (...) | Boolean | `IF([Departure Advance Notice >= Min. Advance Notice]=TRUE AND [Continuous Coverage Between Advance Notice > Min. Advance Notice and Advance Notice <= Min. Advance Notice Predictions]=FALSE) THEN FALSE ELSEIF([Departure Advance Notice >= Min. Advance Notice]=TRUE AND [Continuous Coverage Between Advance Notice > Min. Advance Notice and Advance Notice <= Min. Advance Notice Predictions]=TRUE AND [Prediction Gaps with Min. Advance Notice?]=FALSE) THEN TRUE ELSEIF([Departure Advance Notice >= Min. Advance Notice]=TRUE AND [Continuous Coverage Between Advance Notice > Min. Advance Notice and Advance Notice <= Min. Advance Notice Predictions]=TRUE AND [Prediction Gaps with Min. Advance Notice?]=TRUE) THEN FALSE ELSEIF([Departure Advance Notice >= Min. Advance Notice]=FALSE) THEN FALSE END`

### Data Filters
`Prediction Generated after Terminal Departure`=FALSE

### Parameters
Field Name | Description | Type | Default Value |
--- | --- | --- |  --- |
Min. Advance Notice (minutes) | Minimum advance notice value in minutes |  Number (whole) | 0
Single or Multi-Day? | Allows the user the configure whether the report data should be for single service date or span multiple days | String | Single Day
Single Day Service Date Parameter | If Single Day is selected, allows the user to set which service date the report should look at | Date | `[Latest Data Date]`
Multi-Day Start Service Date Parameter | If Multi-Day is selected, allows the user to set which day the report service date range should start with | Date | `[2 Weeks Ago from Latest Data Date]`
Multi-Day End Service Date Parameter | If Multi-Day is selected, allows the user to set which day the report service date range should end with | Date | `[Latest Data Date]`
