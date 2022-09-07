---
layout: post
title: Using Ratatool to Rank Tennis Players
catagories: Ratatool, Big Data, Scala, BigQuery
---

[Ratatool](https://github.com/spotify/ratatool) is a tooling suite for sampling, generating, and comparing data. It contains several tools worth checking out, but this tutorial be focused on using [BigDiffy](https://github.com/spotify/ratatool/tree/master/ratatool-diffy).

The goal of this tutorial is to create a pipeline that ingests historical WTA Tennis matches and player information, map them to a format [BigDiffy](https://github.com/spotify/ratatool/tree/master/ratatool-diffy) can use, run [BigDiffy](https://github.com/spotify/ratatool/tree/master/ratatool-diffy), and analyze the output. We will be creating a simple [Scio](https://github.com/spotify/scio) pipeline in order to properly map and store the data. The code should be fairly intuitive, and there will be instructions on how to run the pipeline. 

This tutorial will require a GCP environment to run the [Scio](https://github.com/spotify/scio) and [Ratatool](https://github.com/spotify/ratatool) pipelines.

# Getting the Data

WTA tennis ranking data can be found [here](https://github.com/JeffSackmann/tennis_wta). The data can be downloaded via `git clone git@github.com:JeffSackmann/tennis_wta.git`. The files we will be using for this tutorial are `wta_players.csv` and `wta_rankings_10s.csv`. `wta_players.csv` has the following schema: layer_id, first_name, last_name, hand, birth_date, country_code. `wta_rankings_10s.csv` has the schema: ranking_date, rank, player, points, tours. 

For convience of the Scio pipeline, we will store the player and ranking data in GCS. Run `gsutil mb -p $PROJECT_ID gs://tennis-wta-csv` and copy the files over via `gsutil cp tennis_wta/wta_players.csv gs://tennis-wta-csv/wta_players.csv` and `gsutil cp tennis_wta/wta_rankings_10s.csv gs://tennis-wta-csv/wta_rankings_10s.csv`.

# Creating the Scio Pipeline

[Scio](https://github.com/spotify/scio) is a [Scala](https://www.scala-lang.org/) API for [Apache Beam](https://beam.apache.org/) and [Google Cloud Dataflow](https://cloud.google.com/dataflow). We will be writing a simple Scio pipeline to map our tennis match data into [Avro](https://avro.apache.org/), which is a file format that BigDiffy understands. 

In order to create our Scio pipeline, run `sbt new spotify/scio.g8` and enter `yes` to the `DataflowRunner [yes/NO]:` prompt, all other prompts can be left as default. This will create our pipeline under the `scio-job` directory. 

## Pipeline Logic

Since Scio CSV parsing isn't included in the template, update the `build.sbt` and add 
```scala
"com.spotify" %% "scio-extra" % scioVersion,
```
to the `libraryDependencies ++= Seq(...)` list.

Replace the contents of `WordCount.scala` with 

```scala
package example

import com.spotify.scio._
import com.spotify.scio.avro.types.AvroType
import com.spotify.scio.extra.csv._
import com.spotify.scio.values.SCollection
import kantan.csv._

import scala.util.Try

// type to represent a row of wta_players.csv
case class Player(playerId: String, firstName: String, lastName: String)
// type to represent a row of wta_rankings_10s.csv
case class Ranking(rankingDate: String, rank: String, playerId: String, points: String)

// type to represent a PlayerRank Avro record
@AvroType.toSchema
case class PlayerRankingOutput(playerId: String, firstName: String, lastName: String,
                               rankingDate: String, rank: Long, points: Long,
                               playerIdMonth: String)

object WordCount {
  def main(cmdlineArgs: Array[String]): Unit = {
    // parse CLI arguments
    val (sc, args) = ContextAndArgs(cmdlineArgs)
    val year = args("year")
    val wtaPlayersInput = args("wtaPlayerInput")
    val wtaRankingsInput = args("wtaRankingsInput")
    val output = args("output")

    // read wta_players.csv and key by playerId
    val playerCsvs: SCollection[(String, Player)] = {
      implicit val decoder: HeaderDecoder[Player] =
        HeaderDecoder.decoder("player_id", "name_first", "name_last")(Player.apply)
      sc.csvFile(wtaPlayersInput).map(p => (p.playerId, p))
    }

    // read wta_rankings_10s.csv and key by playerId
    val rankingCsvs: SCollection[(String, Ranking)] = {
      implicit val decoder: HeaderDecoder[Ranking] =
        HeaderDecoder.decoder("ranking_date", "rank", "player", "points")(Ranking.apply)
      sc.csvFile(wtaRankingsInput).map(r => (r.playerId, r))
    }
    // join on playerId
    val joined: SCollection[(String, (Player, Ranking))] = playerCsvs
      .join(rankingCsvs)
    
    joined
      // map each row to a PlayerRankingOutput avro       
      .flatMap {
        case (playerId, (player, ranking)) =>
          val month = ranking.rankingDate.substring(4, 6)
          Try(PlayerRankingOutput(
            playerId,
            player.firstName,
            player.lastName,
            ranking.rankingDate,
            ranking.rank.toLong,
            ranking.points.toLong,
            // key by playerId_firstName_lastName_matchMonth
            // this is the column that BigDiffy will use to find player match
            // results across two datasets
            s"${playerId}_${player.firstName}_${player.lastName}_$month"
          )).toOption
      }
      // filter by the year arg
      .filter(playerRanking => playerRanking.rankingDate.startsWith(year))
      // map(...).sampleByKey(...).values.flatten is used to 
      // key by month, then pick a single (arbitrary) match across one month for each month
      .map(playerRanking => (playerRanking.playerIdMonth, playerRanking))
      .sampleByKey(sampleSize = 1)
      .values
      .flatten
      // save as a PlayerRankingOutput avro record
      .saveAsTypedAvroFile(output)

    sc.run().waitUntilDone()
  }
}
```

## Running the pipeline

Before we run big-diffy, we will need to create a dataset in BigQuery for BigDiffy to create tables and write it's output to. We can do this through the CLI via `gsutil mb -p $PROJECT_ID gs://tennis-wta-avro`.

Additionally, we will need a service account with Dataflow Worker, Service Account User, BigQuery Job User, BigQuery Data Owner, BigQuery User, and Storage Creator + Viewer permissions.

Also, be sure to enable the Dataflow API for your project if you haven't already.

We can build the JAR with `sbt stage` and run it via:

```bash
 sbt "runMain example.WordCount
    --project=[PROJECT] --runner=DataflowRunner --zone=us-east1-b --region=us-east1
    --serviceAccount=[SERVICE_ACCOUNT]
    --wtaPlayerInput=gs://tennis-wta-csv/wta_players.csv --wtaRankingsInput=gs://tennis-wta-csv/wta_rankings_10s.csv
    --year=2010
    --output=gs://tennis-wta-avro/wtaRankingsAvro/2010/"
```

since we need another year to compare the data, run the job again with a different year.

```bash
 sbt "runMain example.WordCount
    --project=[PROJECT] --runner=DataflowRunner --zone=us-east1-b --region=us-east1
    --serviceAccount=[SERVICE_ACCOUNT]
    --wtaPlayerInput=gs://tennis-wta-csv/wta_players.csv --wtaRankingsInput=gs://tennis-wta-csv/wta_rankings_10s.csv
    --year=2011
    --output=gs://tennis-wta-avro/wtaRankingsAvro/2011/"
```

## Inspecting the data

The data should be inspectable via [avro-tools](https://avro.apache.org/docs/current/gettingstartedjava.html). Simply pull down one of the avro files with `gsutil cp gs://tennis-wta-avro/wtaRankingsAvro/2010/part-00000-of-00001.avro .` and inspect it with `avro-tools tojson part-00000-of-00001.avro | less` 


# Installing Ratatool

Instructions for installing Ratatool can be found [here](https://github.com/spotify/ratatool#usage). If you're on Mac, there is a Homebrew tap:

```bash
brew tap spotify/public
brew install ratatool
ratatool
```

Otherwise download the [release](https://github.com/spotify/ratatool/releases) jar and run it:

```bash
wget https://github.com/spotify/ratatool/releases/download/v0.3.10/ratatool-cli-0.3.10.tar.gz
tar xvf ratatool-cli-0.3.10.tar.gz
ratatool-cli-0.3.10/bin/big-diffy
```

# Running Ratatool

```bash
bq --project_id=[PROJECT] --location=US mk -d --default_table_expiration 3600 ratatool_wta_output
```
then we can run BigDiffy via

```bash
ratatool-cli-0.3.10/bin/big-diffy --input-mode=avro --output-mode=bigquery --key=playerIdMonth --lhs=gs://tennis-wta-avro/wtaRankingsAvro/2010/\*.avro --rhs=gs://tennis-wta-avro/wtaRankingsAvro/2011/\*.avro --output=ratatool_wta_output.tennis-wta-bigdiffy --project=[PROJECT] --runner=DataflowRunner --region=[REGION] --serviceAccount=[SERVICE_ACCOUNT]
```

# Analyzing the results

To inspect the Ratatool output, navigate to your BigQuery dataset in the GCP console. In there, you'll see three tables that Ratatool has created: `tennis-wta-bigdiffy_fields`, `tennis-wta-bigdiffy_global`, and `tennis-wta-bigdiffy_keys`. 

## tennis-wta-bigdiffy_keys

`tennis-wta-bigdiffy_keys` contains information on the delta's for each key in the 2010 and 2011 datasets. The `key` column corrosponds to that players WTA ID, first name, last name, and month the match took place (seperated by underscores).

 We will see `MISSING_LHS` in the `diffType` column if a player didn't play for that month in 2010, but did play for that month in 2010, and vis-versa for `MISSING_RHS`. A `diffType` of `SAME` means that that player had the same score across both years for that month's match, and a `diffType` of `DIFFERENT` means that player scored differently for that month's match in 2010 and 2011. 

 The `delta.field` column corrosponds to the field Ratatool is comparing. In this case, that column will either be `rank`, corresponding to that player's WTA ranking at the time, and `points`, corresponding to the number of points that player scored. 

 The `delta.left` and `delta.right` correspond to the value for the `delta.field` for 2010 (left) and 2011 (right).

 The `delta.delta.deltaValue` column shows the difference in value from `delta.left` and `delta.right`, so if the `delta.field` is `rank`, the id is `200079_Kim_Clijsters_02`, `delta.left` and `delta.right` are `17` and `1`, and `delta.delta.deltaValue` is `-16`, then that means that Kim Clijsters rank rose from `17` in `2010` to `1` in `2011`. 

 At this point, we can do further analysis in BigQuery by sorting players by the greatest increase in ranking from 2010 and 2011 via:

```sql
SELECT * FROM `[PROJECT].ratatool_wta_output.tennis-wta-bigdiffy_keys` 
WHERE diffType = 'DIFFERENT'
ORDER BY delta.delta.deltaValue asc
LIMIT 1000
```

## tennis-wta-bigdiffy_global

`tennis-wta-bigdiffy_global` contains global stats such as `numTotal`, `numSame`, `numDiff`, `numMissingLhs`, and `numMissingRhs`. 

## tennis-wta-bigdiffy_fields

`tennis-wta-bigdiffy_fields` contains count, min, max, count, mean, variance, standard deviation, skewness, and kurtosis for the rank, points, and rankingDate fields across both datasets.