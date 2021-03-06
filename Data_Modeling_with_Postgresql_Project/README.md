# Introduction to Data Modelling 
I have joined Udacity Data Engineer Nanodegree. The program consists of many real-world projects. In the project, I have the role of a Data Engineer at a fabricated data streaming company called “Sparkify”. It is a startup and wants to analyze the data they’ve been collecting on songs and user activity on their new music streaming app. The analytics team is particularly interested in understanding what songs users are listening to. Currently, they don’t have an easy way to query their data, which resides in a directory of JSON logs on user activity on the app, as well as a directory with JSON metadata on the songs in their app.

They’d like a data engineer to create a Postgres database with tables designed to optimize queries on song play analysis and bring me on the project. My role is to create a database schema and ETL pipeline for this analysis. I’ve tested my database and ETL pipeline by running queries given to me by the analytics team from Sparkify and compare my results with their expected results.

# Project Description
In this project, I’ve applied what I’ve learnt on data modeling with Postgres and build an ETL pipeline using Python. I’ve defined fact and dimension tables for a star schema for a particular analytic focus and written an ETL pipeline that transfers data from files in two local directories into these tables in Postgres using Python and SQL.

# Song Dataset
The first dataset is a subset of real data from the Million Song Dataset. Each file is in JSON format and contains metadata about a song and the artist of that song. The files are partitioned by the first three letters of each song’s track ID. Below is what a single song file looks like.

```
{"num_songs": 1, "artist_id": "ARJIE2Y1187B994AB7", "artist_latitude": null, "artist_longitude": null,
"artist_location": "", "artist_name": "Line Renaud", "song_id": "SOUPIRU12A6D4FA1E1",
"title": "Der Kleine Dompfaff", "duration": 152.92036, "year": 0}
```

# Log Dataset
The second dataset consists of log files in JSON format generated by this event simulator based on the songs in the dataset above. These simulate activity logs from a music streaming app based on specified configurations.
The log files in the dataset you’ll be working with are partitioned by year and month. Below is what a single log file looks like this when opened in csv `(this is to see the data in the file)`.


![Screenshot (152)](https://user-images.githubusercontent.com/55639062/78468769-c404ff80-7712-11ea-82e2-3501b19712ad.png)


# The schema for the Song Play Analysis
Using the song and log datasets, I’ve created a star schema optimized for queries on song play analysis. This includes the following tables.

## Fact Table
**songplays —** records in log data associated with song plays i.e. records with page `NextSong`
*songplay_id, start_time, user_id, level, song_id, artist_id, session_id, location, user_agent*

## Dimension Tables
**users —** users in the app
*user_id, first_name, last_name, gender, level*

**songs —** songs in the music database
*song_id, title, artist_id, year, duration*

**artists —** artists in the music database
*artist_id, name, location, latitude, longitude*

**time —** timestamps of records in songplays broken down into the specific unit
*start_time, hour, day, week, month, year, weekday*

I added constrains and condition on the columns when creating the relational database. Below is the schema diagram for `Sparkify Database`

![Screenshot (148)](https://user-images.githubusercontent.com/55639062/78468855-5e654300-7713-11ea-835a-54c1f0cdf048.png)

# Project Template
In addition to the data files, the project workspace includes five files:
1. `test.ipynb` displays the first few rows of each table to check the database.
1. `create_tables.py` drops and creates the tables. I run this file to reset my tables before each time I run my ETL scripts.
1. `etl.ipynb` reads and processes a single file from song_data and log_data and loads the data into the tables. This notebook contains a detailed explanation of the ETL process for each of the tables.
1. `etl.py` reads and processes files from song_data and log_data and loads them into the tables.
1. `sql_queries.py` contains all the SQL queries, and is imported into the last three files above.

# Create Tables
To create database and all the table used above, I have done the following steps
1. Write `CREATE` statements in `sql_queries.py` to create each table
1. Write `DROP` statements in `sql_queries.py` to drop each table if it exists.
1. Run `create_tables.py` to create the database and tables every time a change was made. To run this I made sure to `Restart Kernel`
1. Run `test.ipynb` to confirm the creation of the tables with the correct columns

#### Here is the function that create table in the `create_table.py` file and catches error if any.

```python
def create_tables(cur, conn):
    """
    Creates each table using the queries in `create_table_queries` list. 
    """
    try:

        for query in create_table_queries:
            cur.execute(query)
            conn.commit()

        print("Table(s) created succussfully")
    except Exception as e:
        print(e)
        
```

#### In sql_queries.py, I have written the queries that perform drops, creates and inserts on all tables in database. Here is the query that drops and creates songplays table.
`Note: This query will give error if it is executed before creating other table because of the constrains`

```python
# DROP TABLE
songplay_table_drop = "DROP TABLE IF EXISTS songplays"

# CREATE TABLE
songplay_table_create = """CREATE TABLE IF NOT EXISTS songplays(
                        songplay_id SERIAL PRIMARY KEY,
                        start_time BIGINT NOT NULL, 
                        user_id INT NOT NULL, 
                        level VARCHAR, 
                        song_id VARCHAR, 
                        artist_id VARCHAR, 
                        session_id int, 
                        location VARCHAR, 
                        user_agent VARCHAR,
                        FOREIGN KEY (song_id) REFERENCES songs(song_id),
                        FOREIGN KEY (user_id) REFERENCES users(user_id),
                        FOREIGN KEY (artist_id) REFERENCES artists(artist_id),
                        FOREIGN KEY (start_time) REFERENCES time(start_time)
                    );"""

```

You can see if the tables have been created with the columns defined from test.ipynb.


# Build ETL Pipeline
To develop the ETL processes, I used `etl.ipnyb` as prototype, using the first song file and first file to understand the data and the data extraction process `(check etl.ipnyb)`. After successfully making the prototype, I worked on `etl.py` to create the script the does the ETL pipeline.

In etl.py, there is a function called `process_data`. The function take in the filepaths, iterates over file (including the sub-directories), get all `.json extension`  and then processes it.
Below is the function

```python
def process_data(cur, conn, filepath, func):
    """
    Parameters:
        cur: This holds the data retrieved from database

        conn: This holds the connection made to the database

        filepath: This holds the directory to the folder to be considered.

        func: This holdd functions

    Function:
        To process all song files in the filepath and extract specific information
        from all the file involved
    """

    # get all files matching extension from directory
    all_files = []
    for root, dirs, files in os.walk(filepath):
        files = glob.glob(os.path.join(root,'*.json'))
        for f in files :
            all_files.append(os.path.abspath(f))

    # get total number of files found
    num_files = len(all_files)
    print('{} files found in {}'.format(num_files, filepath))

    # iterate over files and process
    for i, datafile in enumerate(all_files, 1):
        func(cur, datafile)
        conn.commit()
        print('{}/{} files processed.'.format(i, num_files))
       
```       

# Process `song_data`
So from the `process_data function`, I handled the `song_data` first and I created from it the songs and artists dimensional tables. I’ve built `process_song_file` function to perform the following tasks:
* First, get a list of all song JSON files in data/song_data
* Read the song file
* Extract Data for Songs Table and Insert Data into Songs Table
* Extract Data for Artists Table and Insert Data into Artists Table

## Extract Data for Songs Table
To extract data for Songs table, the following tasks are done:
* Select columns for song ID, title, artist ID, year, and duration
* Use `song_col.values` to select just the values from the DataFrame 
* Convert the array to a list and set it to `song_data` (check in the code, etl.py)

## Insert Record into Song Table
After I extracted the data in to song_data dataframe, I inserted it into the song table by writing `song_table_insert query` in `sql_queries.py`.

#### Here is how the `song_table_insert query` looks like
```python
song_table_insert = """INSERT INTO songs
                    VALUES (%s, %s, %s, %s, %s)
                    ON CONFLICT
                    DO NOTHING;
                    """
```     
#### Here is the code that inserts record in the songs table 
```python
df = pd.read_json(filepath, lines=True)

# insert song record
song_col =  df[[ 'song_id', 'title', 'artist_id', 'year', 'duration']]
song_col =  song_col.values

for data in song_col:
    song_data = []
    for value in data:
        song_data.append(value)

cur.execute(song_table_insert, song_data)
```
## Extract Data for Artists Table and Insert Record into Artist Table
The extraction process is similar to that for Songs table, but I selected columns artist ID, name, location, latitude, and longitude.

# Process `log_data`
I’ve performed ETL on the log_data, to create the `time` and `users` dimensional tables and the `songplays` fact table.

## Extract Data for Time Table and Insert Records into Time Table
For this, I filtered the records by `NextSong` 

```python
filt = (df['page'] == "NextSong")
df = df[filt]
```

After filtering, I extracted the timestamp, hour, day, week of year, month, year, and weekday from the `ts` column and set `time_data` to a list containing these values in order. I’ve also specified labels for these columns and set to `column_labels`. I've also created a dataframe, time_df, containing the time data for this file by combining column_labels and time_data

```python
ts = df['ts'] 
time_data = []

for what in ts:
    data = convert_ts(what)
    time_data.append(data)
    
column_labels = ('time_stamp', 'hour', 'day', 'week_of_year', 'month', 'year', 'weekday')    

# Creating time_df
time_df = pd.DataFrame(time_data, columns=column_labels)

```

To do the extraction, I created a function called `convert_ts` to perform the extraction process. Here is the function

```python
def convert_ts(ts):
    t = datetime.fromtimestamp(ts/1000)
    try:
        hour = t.hour
        day =  t.day
        week_of_year = t.isocalendar()[1]
        month = t.month
        year = t.year
        weekday =  t.weekday()

        data = (ts, hour, day, week_of_year, month, year, weekday)

    except Exception as e:
        print(e)

    return data
    
```    

# Extract Data for Users Table and Insert Records into Users Table
Data extraction for users table is easy, I just selected columns for user ID, first name, last name, gender, and level and set to user_df and inserted records for the users in this log file into the users table.

## Extract Data into Songplays Table
This one is a little more complicated since the information from the `songs` table, `artists` table and original log file are all needed for the `songplays` table. Since the log file does not specify an ID for either the song or the artist, I needed to get the song ID and artist ID by querying the `songs` and `artists` tables to find matches based on the song title, artist name, and the song’s duration time.

To get data to songplays table is a test of `SQL knowledge`. So I will start with the insert query from the `sql_queries.py`, I joined songs and artists table using the conditions above to find songs that matches.

```python
# FIND SONGS
song_select = ("""
                SELECT 
                    s.song_id, 
                    a.artist_id 
                FROM 
                    songs s
                INNER JOIN 
                    artists a ON a.artist_id = s.artist_id
                WHERE s.title = (%s) AND a.artist_name = (%s) AND s.duration = (%s)
""")
```
#### Here is the code that carried out the insertion proess:
```python
for index, row in df.iterrows():
        
        # get songid and artistid from song and artist tables
        cur.execute(song_select, (row.song, row.artist, row.length))
        results = cur.fetchone()
        
        if results:
            songid, artistid = results
        else:
            songid, artistid = None, None

        # insert songplay record
        songplay_data = (row.ts, row.userId, row.level, songid, artistid, row.sessionId, row.location, row.userAgent)
        cur.execute(songplay_table_insert, songplay_data)
        
```

# Conclusion
Successfully, I've created a Postgres database with tables designed to optimize queries on song plays analysis. The team can easily analyze the data they’ve been collecting on songs and user activity on their new music streaming app and can easily understand what songs users are listening to.
    
