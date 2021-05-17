# datapump
Pump time-series data into the CKAN datastore using a simple filesystem-based queueing system.

Requires: Python 3.8

Installation:
=============

Linux, Mac, Windows Powershell:
```
python3 -m venv datapumpenv
. datapumpenv/bin/activate
cd datapumpenv
git clone https://github.com/dathere/datapump.git
cd datapump
pip install -r requirements.txt
```

Windows CMD:
```
python3 -m venv datapumpenv
datapumpvenv\Scripts\activate
cd datapumpenv
git clone https://github.com/dathere/datapump.git
cd datapump
pip install -r requirements.txt
```

Usage:
======

Linux, Mac, Windows Powershell:
```
. datapumpenv/bin/activate
cd datapumpenv/datapump
python datapump.py --config datapump.ini
```

Windows CMD:
```
datapumpenv\Scripts\activate
cd datapumpenv\datapump
python datapump.py --config datapump.ini
```

Command line parameters:
------------------------

```
python datapump.py --help
Usage: datapump.py [OPTIONS]

  Pumps time-series data into CKAN using a simple filesystem-based queueing system.

Options:
  --inputdir PATH      The directory where the job files are located.  [default: ./input]
  --processeddir PATH  The directory where successfully processed inputfiles are moved.  [default: ./processed]
  --problemsdir PATH   The directory where unsuccessful inputfiles are moved.  [default: ./problems]
  --datecolumn TEXT    The name of the datetime column.  [default: DateTime]
  --dateformats TEXT   List of dateparser format strings to try one by one. See https://dateparser.readthedocs.io
                       [default: %y-%m-%d %H:%M:%S, %y/%m/%d %H:%M:%S, %Y-%m-%d %H:%M:%S, %Y/%m/%d %H:%M:%S]

  --host TEXT          CKAN host.  [required]
  --apikey TEXT        CKAN api key to use.  [required]
  --verbose            Show more information while processing.
  --debug              Show debugging messages.
  --logfile PATH       The full path of the main log file.  [default: ./datapump.log]
  --config FILE        Read configuration from FILE.
  --version            Show the version and exit.
  --help               Show this message and exit.
```

Note that parameters are parsed and processed in priority order - through environment variables, a config file, or through the command line interface.

Environment variables should be all caps and prefixed with `DATAPUMP_`, for example:

```
export DATAPUMP_APIKEY="MYCKANAPIKEY"
export DATAPUMP_HOST="https://ckan.example.com"
```

Job JSON
--------

The input directory is scanned for `*-job.json` files in date descending order, executing each job per the JSON configuration.

For example:

```
{
	"InputFile": "./samples/zone1_airquality_*.csv",
	"TargetOrg": "etl-test",
	"TargetPackage": "iot-test",
	"TargetResource": "air-quality",
	"PrimaryKey": "DateTime,Sensor_id",
	"Dedupe": "last",
	"$comment": "deduplicate rows, retaining the last duplicate value as per the primary key definition",
	"Truncate": false,
  "$comment": "truncate is false, DO NOT drop existing rows in the targetResource. Do an upsert",
	"Stats": [{
			"Kind": "descriptive"
		},
		{
			"Kind": "mode"
		},
		{
			"Kind": "H",
			"$comment": "H - resample at hourly frequency",
			"GroupBy": "Sensor_id",
			"DropColumns": "LAT,LONG",
      "$comment": "It doesn't make sense to resample lat, long. Drop these columns from the resampled data",
		},
		{
			"Kind": "D",
			"$comment": "D - resample at daily frequency",
			"GroupBy": "Sensor_id",
			"DropColumns": "LAT,LONG"
		}
	]
}
```

Note the `Dedupe` attribute specifies if datapump should automatically handle duplicate rows using the `PrimaryKey` attribute to determine duplication.

It can be set to `first`, `last` or ''.
`first` : Drop duplicates except for the first occurrence. - `last` : Drop duplicates except for the last occurrence. - '' : Do not drop duplicates.

Datapump can also compute descriptive statistics and resamples time-series data (e.g. converting per-minute data to hourly data).

For descriptive statistics, it creates a table with the `-stat` suffix. It computes:
 * count - count number of non-NA/null observations
 * unique - unique values
 * top - most common value
 * freq - most common value's frequency
 * mean - mean
 * std - standard deviation
 * min - minimum
 * 25% - 25% percentile
 * 50% - 50% percentile
 * 75% - 75% percentile
 * max - maximum

It can also compute the mode (the most common value of each column) for a dataset in a separate table ending with a `-mode` suffix.

Finally, it can also resample a time-series dataset using any valid Pandas resampling offset.

See https://pandas.pydata.org/pandas-docs/stable/user_guide/timeseries.html#dateoffset-objects and 
https://pandas.pydata.org/pandas-docs/stable/user_guide/timeseries.html#offset-aliases valid resampling offset values.

To do so, just specify the desired offset for the `Kind` attribute.
