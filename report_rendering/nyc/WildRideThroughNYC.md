
```
@file:MavenRepository("bintray-plugins", "http://jcenter.bintray.com")

@file:DependsOnMaven("com.github.holgerbrandl:kravis:0.5")
@file:Suppress("PropertyName")

//@file:DependsOnMaven("ml.dmlc:xgboost4j:0.80")


import krangl.*
import krangl.experimental.oneHot
import kravis.*
import kravis.OrderUtils.reorder
import ml.dmlc.xgboost4j.java.DMatrix
import ml.dmlc.xgboost4j.java.XGBoost
import java.io.File
import java.lang.Math.*
import java.time.DayOfWeek
import java.time.LocalDateTime
import java.time.Month
import java.time.format.DateTimeFormatter
```

# A wild ride through NYC with Kotlin
Predict taxi trip durations in NYC. For detail see https://www.kaggle.com/c/nyc-taxi-trip-duration

```
val dataRoot = File("/Users/brandl/Desktop/taxi_data")

dataRoot
    .listFiles()
    .forEach { println(it) }

operator fun File.div(fileName: String) = this.resolve(fileName)

var allTrainData = DataFrame.readCSV(dataRoot / "train.csv")
var trainData = allTrainData.sampleFrac(0.3)
```

Live@KC Explore structure and differences between test and training data

```
trainData
trainData.head()
trainData.schema()
```

Live@KC Finetune shell experience

```
PRINT_MAX_WIDTH = 80

trainData
trainData.schema()
```

Live@KC Compare with test data

```
var testData = DataFrame.readCSV(dataRoot / "test.csv")
testData.schema()
```

Live@KC How to efficiently access data-frames?

```
trainData["vendor_id"]
```

Define columns names as fields for better completion

```
val id: String = "id"
val vendor_id: String = "vendor_id"
val pickup_datetime: String = "pickup_datetime"
val dropoff_datetime: String = "dropoff_datetime"
val passenger_count: String = "passenger_count"
val pickup_longitude: String = "pickup_longitude"
val pickup_latitude: String = "pickup_latitude"
val dropoff_longitude: String = "dropoff_longitude"
val dropoff_latitude: String = "dropoff_latitude"
val store_and_fwd_flag: String = "store_and_fwd_flag"
val trip_duration: String = "trip_duration"

val distance = "distance"
val month = "month"
val wday = "wday"
val hour = "hour"
val work = "work"


trainData[vendor_id]
```

## Feature Engineering

```
fun prepareFeatures(trainData: DataFrame): DataFrame {
    var trainData = trainData
    val datePattern = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")

    // LIVE@KC explore date format
    val aDate = trainData[pickup_datetime].asStrings().first()
    LocalDateTime.parse(aDate!!)

    trainData = trainData.addColumns(
        pickup_datetime `=` { it[pickup_datetime].map<String> { LocalDateTime.parse(it, datePattern) } }
    )
    //https://stackoverflow.com/questions/639695/how-to-convert-latitude-or-longitude-to-meters
    val coordDistance = fun(lon1: Double, lat1: Double, lon2: Double, lat2: Double): Double {
        val R = 6378.137                                // radius of earth in Km
        val dLat = (lat2 - lat1) * PI / 180
        val dLon = (lon2 - lon1) * PI / 180
        val a = pow(sin((dLat / 2)), 2.0) + cos(lat1 * PI / 180) * cos(lat2 * PI / 180) * pow(sin(dLon / 2), 2.0)
        val c = 2 * atan2(sqrt(a), sqrt(1 - a))
        val d = R * c
        return (d * 1000)                           // distance in meters
    }

    trainData = trainData.addColumn(distance) {
        df.rows.map { row ->
            coordDistance(
                row[pickup_longitude] as Double,
                row[pickup_latitude] as Double,
                row[dropoff_longitude] as Double,
                row[dropoff_latitude] as Double
            )
        }
    }

    trainData = trainData.addColumns(
        // month `=` { it[pickup_datetime].asType<LocalDateTime>().mapNonNull { it.month } },
        month `=` { it[pickup_datetime].map<LocalDateTime>() { it.month } },
        wday `=` { it[pickup_datetime].asType<LocalDateTime>().mapNonNull { it.dayOfWeek } },
        hour `=` { it[pickup_datetime].asType<LocalDateTime>().mapNonNull { it.hour } },
        work `=` { it[hour].map<Int> { (8..18).contains(it) } }
    )

    return trainData
}


trainData = prepareFeatures(trainData)
trainData.schema()

//trainData.writeTSV(File("trainDataTmp.txt"))
//trainData = DataFrame.readTSV(File("trainDataTmp.txt"))


//fun DataFrame.cleanup(): DataFrame = filter { it[trip_duration] lt 22 * 3600.0 }
//        .filter { it[distance] gt 0.0 }
//        .filter { it[trip_duration] gt 10.0 }
//        .filter { it["speed"] lt 100.0 }
```

## Data Visualisation
LIVE@KC Analyze passenger counts per trip

```
trainData.plot(x = passenger_count.asDiscreteVariable)
    .geomBar()
    .xLabel("# Passengers")
    .show()
```

LIVE@KC Analyze overall distribution of the trip duration

```
//trainData.plot(trip_duration).geomHistogram()
trainData.filter { it[trip_duration] lt 1000 }
    .plot(x = trip_duration)
    .geomHistogram()
    .show()
```

Is it really NYC?

```
trainData.plot(x = pickup_longitude, y = pickup_latitude).geomPoint(alpha = .1)

fun DataFrame.constrainCoord(): DataFrame = filter {
    with(it[pickup_longitude]) { (this gt -74.05) AND (this lt -73.75) } AND
        with(it[pickup_latitude]) { (this gt 40.6) AND (this lt 40.9) }
}


trainData
    .constrainCoord()
    .plot(x = pickup_longitude, y = pickup_latitude)
    .geomPoint(alpha = .1, size = .3)
    .show()

trainData
    .constrainCoord()
    .plot(x = pickup_longitude, y = pickup_latitude)
    .geomBin2D(bins = 90)
    .show()
```

Correlate distance with trip duration (is it a promising predictor?)

```
trainData
    .sampleN(1e4.toInt())
    .plot(distance, trip_duration)
    .geomPoint()
    .scaleYLog10().scaleXLog10()
    .show()
```

try again  but with binning

```
trainData
    .plot(distance, trip_duration)
    .geomBin2D()
    .scaleYLog10().scaleXLog10()
    .show()
```

Speed analysis

```
trainData = trainData.addColumn("speed") { it[distance] / it[trip_duration] * 3.6 }


trainData.filter { (it["speed"] gt 2) AND (it["speed"] lt 1e2) }
    .plot("speed")
    .geomHistogram(fill = RColor.red, bins = 50)
    .labs(x = "Average speed [km/h] (direct distance)")
    .show()
```

Live@KC Extract field for speed property
Speed analysis by day and hour

```
trainData.addColumn(wday) { it[wday].map<DayOfWeek> { it.value } }
```

Live@KC Visualize reasonable (<40) speeds per week day

```
trainData
    .filter { it["speed"] lt 40.0 }
    .plot(wday, "speed").geomBoxplot()
    .show()
```

Live@KC Visualize reasonable speeds also per hour

```
trainData.filter { it["speed"] lt 40.0 }
    .plot(hour.asDiscreteVariable, "speed").geomBoxplot()
    .show()
```

Live@KC wday x hours x medians

```
trainData
//    .filter { (it["speed"] lt 40.0) AND (it[trip_duration] lt 3600) }
    .groupBy(wday, hour)
    .summarize("median_speed" `=` { it["speed"].median() })
    .addColumn("wday_order") { it[wday].map<DayOfWeek> { it.value } }
    .plot(hour, reorder(wday, "wday_order"), fill = "median_speed")
    .geomTile()
    .labs(x = "Hour of the day", y = "Day of the week")
    .show()
```

## Build a predictive model with xgboost

```
// https://youtrack.jetbrains.net/issue/KT-24491 and // https://github.com/khud/sparklin/issues/34)

// LIVE@KC Create helper to split train and validation data (inkl refac into extension method)
fun DataFrame.splitTrainTest(splitProportion: Double = 0.7) = shuffle().run {
    val splitter = (splitProportion * nrow).toInt()
    slice(1..splitter) to slice((splitter + 1)..nrow)
}

//val (train, validation) = trainData.splitTrainTest(splitProportion = 0.4)
// does not work yet in kshell

val dataSplit = prepareFeatures(allTrainData).splitTrainTest()
val trainMatDf = dataSplit.first
var valMatDf = dataSplit.second


fun DataFrame.selectPredictors(): DataFrame = select(
    passenger_count, pickup_longitude, pickup_latitude, dropoff_longitude, dropoff_latitude, distance, month, wday, hour, work
).oneHot<Month>(month)
    .oneHot<DayOfWeek>(wday)
    .addColumn(work) { rows.map { if (it[work] as Boolean) 1 else 0 } }


fun DataFrame.buildTrainMatrix(responseVariable: String = trip_duration): DMatrix {
    val x = selectPredictors().toFloatMatrix()
    val xLong: FloatArray = x.reduce { left, right -> left + right }
    val y = this[responseVariable].asDoubles().map { it!!.toFloat() }.toFloatArray()
    return DMatrix(xLong, nrow, ncol - 1).apply { label = y }
}

// visualize feature matrix
//trainMatDf.head(2000)
//    .addColumn("id"){rowNumber}
//    .gather("predictor", "value", columns= { except("id")})
//    .plot("predictor", "id", fill="value")
//    .geomTile()
//    .theme(axisTitleX=ElementTextBlank())


//trainMatDf.toFloatMatrix().reduce { left, right -> left + right }.size
// Construct the training input for XGBoost
val trainMat = trainMatDf.buildTrainMatrix()


val params = hashMapOf<String, Any>().apply {
    put("objective", "reg:linear")
    put("eval_metric", "rmse")
}


val watches = hashMapOf<String, DMatrix>().apply {
    put("train", trainMat)
    put("validation", valMatDf.buildTrainMatrix())
}

// number of boosting iteration =3 would just build a simple 2-step function model
val nround = 10
val booster = XGBoost.train(trainMat, params, nround, watches, null, null)
```

How does our model look like

```
booster
```

Predict trip duration

```
//var predicts = booster.predict(trainMat)
var predicts = booster.predict(valMatDf.buildTrainMatrix())
```

Unwrap result

```
val predictUnwrapped = predicts.map { it.first() }

predicts.size
```

Combine predictions with ground truth

```
val predTripDurcation = "predicted_trip_duration"
valMatDf = valMatDf.addColumn(predTripDurcation) { predictUnwrapped }
valMatDf.schema()
```

LIVE@KC explore correlation between ground truth and predicted duration

```
valMatDf.sampleN(10000)
    .plot(x = trip_duration, y = predTripDurcation)
    .geomPoint(alpha = .1)
    .scaleXLog10()
    .scaleYLog10()
    .show()


// explore feature importance
//booster.getFeatureScore(null)
```

## Prepare submission file

```
fun DataFrame.buildTestMatrix(): DMatrix {
    val x = toFloatMatrix()
    val xLong: FloatArray = x.reduce { left, right -> left + right }
    return DMatrix(xLong, nrow, ncol)
}


val testFeat = prepareFeatures(testData).selectPredictors().buildTestMatrix()

var testPrediction = booster.predict(testFeat).map { it.first() }
```

LIVE@KC create final submission file

```
val kaggleSubmission = testData
    .addColumn("trip_duration") { testPrediction }
    .select("id", "trip_duration")
```

Final Results schema

```
kaggleSubmission.schema()
kaggleSubmission.writeCSV(File("kotlin4kaggle.csv"))

println("Finished first (out of N>>1) kaggle iteration using kotlin!")
```

submit with
`kaggle competitions submit -c nyc-taxi-trip-duration -f kotlin4kaggle.csv -m "Proof of Concept kernel written in Kotlin"`
