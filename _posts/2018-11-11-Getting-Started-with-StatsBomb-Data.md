---
image: https://raw.githubusercontent.com/statsbomb/open-data/master/img/statsbomb-logo.jpg
---

# Getting Started with Statsbomb Data

## 1) Get the Data

Go to https://github.com/statsbomb/open-data, click on the green 'Clone or download' button and choose download ZIP to your harddrive.
Unpack the zip.

### 2) Take the time to read their terms and conditions.
#### From their github page:
"Whilst we are keen to share data and facilitate research, we also urge you to be responsible with the data. Please register your details on https://www.statsbomb.com/resource-centre and read our User Agreement carefully."

### https://towardsdatascience.com/advanced-sports-visualization-with-pandas-matplotlib-and-seaborn-9c16df80a81b

is a great article to get started using StatsBomb Data and showcases Panda's json_normalize function.<br>
It provides us with a super easy way to take the nested json structure and put it into a nice and orderly DataFrame.

## 3) Process the data and combine it into DataFrames
This will create a DataFrame for all events, one for all Freeze Frames, one for all lineup information, including the minutes played, and one with the information for all Matches

So far I relied on R (FC_rStats made a great series of posts here: https://github.com/FCrSTATS/StatsBomb_WomensData) <br> to process the data, now I can get all the information Statsbomb Data provides with Python, including all the Freeze Frame Information.


```python
import json
import pandas as pd
import numpy as np
from pandas.io.json import json_normalize
from os import listdir
from os.path import isfile, join

'''
Set mypath to your open-data-master/data/ path
'''
mypath =


# EVENTS AND FREEZE-FRAMES
files = [f for f in listdir(mypath+'events/') if isfile(join(mypath+'events/', f))]
try: #if you're on MacOS like I am this file might mess with you, so try removing it
    files.remove('.DS_Store')
except:
    pass

dfs = {}
ffs = {}

for file in files:
    with open(mypath+'events/'+file) as data_file:
        #print (mypath+'events/'+file)
        data = json.load(data_file)
        #get the nested structure into a dataframe
        df = json_normalize(data, sep = "_").assign(match_id = file[:-5])
        #store the dataframe in a dictionary with the match id as key (remove '.json' from string)
        dfs[file[:-5]] = df.set_index('id')    
        shots = df.loc[df['type_name'] == 'Shot'].set_index('id')

        #get the freeze frame information for every shot in the df
        for id_, row in shots.iterrows():
            try:
                ff = json_normalize(row.shot_freeze_frame, sep = "_")
                ff = ff.assign(x = ff.apply(lambda x: x.location[0], axis = 1)).\
                        assign(y = ff.apply(lambda x: x.location[1], axis = 1)).\
                        drop('location', axis = 1).\
                        assign(id = id_)
                ffs[id_] = ff
            except:
                pass

#concatenate all the dictionaries
#this creates a multi-index with the dictionary key as first level
df = pd.concat(dfs, axis = 0)

#split locations into x and y components
df[['location_x', 'location_y']] = df['location'].apply(pd.Series)
df[['pass_end_location_x', 'pass_end_location_y']] = df['pass_end_location'].apply(pd.Series)

#split the shot_end_locations into x,y and z components (some don't include the z-part)
df['shot_end_location_x'], df['shot_end_location_y'], df['shot_end_location_z'] = np.nan, np.nan, np.nan
end_locations = np.vstack(df.loc[df.type_name == 'Shot'].shot_end_location.apply(lambda x: x if len(x) == 3
                                       else x + [np.nan]).values)
df.loc[df.type_name == 'Shot', 'shot_end_location_x'] = end_locations[:, 0]
df.loc[df.type_name == 'Shot', 'shot_end_location_y'] = end_locations[:, 1]
df.loc[df.type_name == 'Shot', 'shot_end_location_z'] = end_locations[:, 2]
events_df = df.drop(['location', 'pass_end_location', 'shot_end_location'], axis = 1)

#concatenate all the Freeze Frame dataframes
ff_df = pd.concat(ffs, axis = 0)


# MATCHES
files = [f for f in listdir(mypath+'matches/') if isfile(join(mypath+'matches/', f))]
try:
    files.remove('.DS_Store')
except:
    pass

matches_dfs = {}
for file in files:
    with open(mypath+'matches/'+file) as data_file:
        #print (mypath+'lineups/'+file)
        data = json.load(data_file)
        #get the nested structure into a dataframe
        df_ = json_normalize(data, sep = "_")
        #store the dataframe in a dictionary with the competition id as key
        matches_dfs[file[:-5]] = df_

matches_df = pd.concat(matches_dfs)


# LINEUPS w Minutes played
files = [f for f in listdir(mypath+'lineups/') if isfile(join(mypath+'lineups/', f))]
try: #if you're on MacOS like I am this file might mess with you, so try removing it
    files.remove('.DS_Store')
except:
    pass


dfs = {}
ffs = {}

for file in files:
    with open(mypath+'lineups/'+file) as data_file:
        #print (mypath+'events/'+file)
        data = json.load(data_file)
        #get the nested structure into a dataframe
        df = json_normalize(data, sep = "_").assign(match_id = file[:-5])
        df_1 = json_normalize(df.lineup.iloc[0], sep = "_").assign(
                team_id = df.team_id.iloc[0],
                team_name = df.team_name.iloc[0],
                match_id = df.match_id.iloc[0])
        df_2 = json_normalize(df.lineup.iloc[1], sep = "_").assign(
                team_id = df.team_id.iloc[1],
                team_name = df.team_name.iloc[1],
                match_id = df.match_id.iloc[1])
        dfs[file[:-5]] = pd.concat([df_1, df_2])

lineups_df = pd.concat(dfs.values())

# get the lengths of matches
match_lengths = events_df.groupby('match_id')['minute'].max()

# get all substitutions
substitutions = events_df.loc[events_df.substitution_outcome_name.notnull(),
                               ['minute', 'player_name', 'substitution_replacement_name']].\
                    reset_index().\
                    drop('id', axis = 1).\
                    rename(columns = {'level_0': 'match_id'}).\
                    set_index(['match_id'])

# assign all minutes played to the lineups_df
a = lineups_df.reset_index().set_index('match_id').assign(minutes_played = match_lengths)

for idx, row in substitutions.iterrows():
    a.loc[(a.index == idx)&(a.player_name == row.player_name), 'minutes_played'] = row.minute
    a.loc[(a.index == idx)&(a.player_name == row.substitution_replacement_name), 'minutes_played'] = \
        a.loc[(a.index == idx)&(a.player_name == row.substitution_replacement_name), 'minutes_played'] - row.minute

lineups_df = a.reset_index().set_index(['match_id', 'index'])
```

### Save the data
HDF Files provide an easy and fast way to store and read larger files.


```python
events_df.to_hdf(mypath+'Statsbomb_Data_df.hdf', key = 'df')
ff_df.to_hdf(mypath+'Statsbomb_Data_ff_df.hdf', key = 'ff_df')
matches_df.to_hdf(mypath+'Statsbomb_Data_matches_df.hdf', key = 'matches_df')
lineups_df.to_hdf(mypath+'Statsbomb_Data_lineups_df.hdf', key = 'lineups_df')
```

### Read the data


```python
df = pd.read_hdf(mypath+'Statsbomb_Data_df.hdf')
ff_df = pd.read_hdf(mypath+'Statsbomb_Data_ff_df.hdf')
matches_df = pd.read_hdf(mypath+'Statsbomb_Data_matches_df.hdf')
lineups_df = pd.read_hdf(mypath+'Statsbomb_Data_lineups_df.hdf')
```


```python
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>50_50_outcome_id</th>
      <th>50_50_outcome_name</th>
      <th>bad_behaviour_card_id</th>
      <th>bad_behaviour_card_name</th>
      <th>ball_receipt_outcome_id</th>
      <th>ball_receipt_outcome_name</th>
      <th>ball_recovery_offensive</th>
      <th>ball_recovery_recovery_failure</th>
      <th>block_deflection</th>
      <th>block_offensive</th>
      <th>...</th>
      <th>type_id</th>
      <th>type_name</th>
      <th>under_pressure</th>
      <th>location_x</th>
      <th>location_y</th>
      <th>pass_end_location_x</th>
      <th>pass_end_location_y</th>
      <th>shot_end_location_x</th>
      <th>shot_end_location_y</th>
      <th>shot_end_location_z</th>
    </tr>
    <tr>
      <th></th>
      <th>id</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="5" valign="top">19714</th>
      <th>85e3649d-96cf-43b0-9327-8f7e847f9c2d</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>35</td>
      <td>Starting XI</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>a9f001b3-0beb-4fe3-9562-eaf5298a69b8</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>35</td>
      <td>Starting XI</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>2b624578-b4b5-4bfd-a8c0-531d9717ac19</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>18</td>
      <td>Half Start</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>e4989789-13c9-48ca-8406-5587f40be36e</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>18</td>
      <td>Half Start</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>226163ce-4a29-45b0-90f5-6eb02594223b</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>...</td>
      <td>30</td>
      <td>Pass</td>
      <td>NaN</td>
      <td>61.0</td>
      <td>41.0</td>
      <td>50.0</td>
      <td>35.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 121 columns</p>
</div>




```python
ff_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>player_id</th>
      <th>player_name</th>
      <th>position_id</th>
      <th>position_name</th>
      <th>teammate</th>
      <th>x</th>
      <th>y</th>
      <th>id</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="5" valign="top">000e60b5-955a-4c75-8874-f8b5e4579abf</th>
      <th>0</th>
      <td>15614</td>
      <td>Sophie Elizabeth Bradley-Auckland</td>
      <td>4</td>
      <td>Center Back</td>
      <td>False</td>
      <td>109.0</td>
      <td>41.0</td>
      <td>000e60b5-955a-4c75-8874-f8b5e4579abf</td>
    </tr>
    <tr>
      <th>1</th>
      <td>15618</td>
      <td>Jasmine Matthews</td>
      <td>5</td>
      <td>Left Center Back</td>
      <td>False</td>
      <td>106.0</td>
      <td>43.0</td>
      <td>000e60b5-955a-4c75-8874-f8b5e4579abf</td>
    </tr>
    <tr>
      <th>2</th>
      <td>15626</td>
      <td>Anke Preuß</td>
      <td>1</td>
      <td>Goalkeeper</td>
      <td>False</td>
      <td>119.0</td>
      <td>43.0</td>
      <td>000e60b5-955a-4c75-8874-f8b5e4579abf</td>
    </tr>
    <tr>
      <th>3</th>
      <td>15629</td>
      <td>Satara Murray</td>
      <td>3</td>
      <td>Right Center Back</td>
      <td>False</td>
      <td>103.0</td>
      <td>27.0</td>
      <td>000e60b5-955a-4c75-8874-f8b5e4579abf</td>
    </tr>
    <tr>
      <th>4</th>
      <td>15616</td>
      <td>Kim Little</td>
      <td>15</td>
      <td>Left Center Midfield</td>
      <td>True</td>
      <td>100.0</td>
      <td>34.0</td>
      <td>000e60b5-955a-4c75-8874-f8b5e4579abf</td>
    </tr>
  </tbody>
</table>
</div>




```python
matches_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>away_score</th>
      <th>away_team_away_team_id</th>
      <th>away_team_away_team_name</th>
      <th>competition_competition_id</th>
      <th>competition_competition_name</th>
      <th>competition_country_name</th>
      <th>data_version</th>
      <th>home_score</th>
      <th>home_team_home_team_id</th>
      <th>home_team_home_team_name</th>
      <th>kick_off</th>
      <th>last_updated</th>
      <th>match_date</th>
      <th>match_id</th>
      <th>match_status</th>
      <th>referee_name</th>
      <th>season_season_id</th>
      <th>season_season_name</th>
      <th>stadium_name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="5" valign="top">37</th>
      <th>0</th>
      <td>0</td>
      <td>746</td>
      <td>Manchester City WFC</td>
      <td>37</td>
      <td>FA Women's Super League</td>
      <td>England</td>
      <td>1.0.3</td>
      <td>0</td>
      <td>971</td>
      <td>Chelsea LFC</td>
      <td>15:00:00.000</td>
      <td>2018-10-16T17:04:34.945</td>
      <td>2018-09-09</td>
      <td>19714</td>
      <td>available</td>
      <td>None</td>
      <td>4</td>
      <td>2018/2019</td>
      <td>Cherry Red Records Fans' Stadium</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0</td>
      <td>967</td>
      <td>Everton LFC</td>
      <td>37</td>
      <td>FA Women's Super League</td>
      <td>England</td>
      <td>1.0.3</td>
      <td>1</td>
      <td>969</td>
      <td>Birmingham City WFC</td>
      <td>15:00:00.000</td>
      <td>2018-09-14T14:42:50.415805</td>
      <td>2018-09-09</td>
      <td>19718</td>
      <td>available</td>
      <td>None</td>
      <td>4</td>
      <td>2018/2019</td>
      <td>Damson Park</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0</td>
      <td>970</td>
      <td>Yeovil Town LFC</td>
      <td>37</td>
      <td>FA Women's Super League</td>
      <td>England</td>
      <td>1.0.3</td>
      <td>4</td>
      <td>974</td>
      <td>Reading WFC</td>
      <td>15:00:00.000</td>
      <td>2018-10-16T17:02:59.817</td>
      <td>2018-09-09</td>
      <td>19716</td>
      <td>available</td>
      <td>None</td>
      <td>4</td>
      <td>2018/2019</td>
      <td>Adams Park</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0</td>
      <td>966</td>
      <td>Liverpool WFC</td>
      <td>37</td>
      <td>FA Women's Super League</td>
      <td>England</td>
      <td>1.0.3</td>
      <td>5</td>
      <td>968</td>
      <td>Arsenal WFC</td>
      <td>13:30:00.000</td>
      <td>2018-10-16T17:03:47.733</td>
      <td>2018-09-09</td>
      <td>19717</td>
      <td>available</td>
      <td>None</td>
      <td>4</td>
      <td>2018/2019</td>
      <td>Meadow Park</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1</td>
      <td>973</td>
      <td>Bristol City WFC</td>
      <td>37</td>
      <td>FA Women's Super League</td>
      <td>England</td>
      <td>1.0.3</td>
      <td>0</td>
      <td>965</td>
      <td>Brighton &amp; Hove Albion WFC</td>
      <td>15:00:00.000</td>
      <td>2018-10-16T17:05:18.428</td>
      <td>2018-09-09</td>
      <td>19715</td>
      <td>available</td>
      <td>None</td>
      <td>4</td>
      <td>2018/2019</td>
      <td>Broadfield Stadium</td>
    </tr>
  </tbody>
</table>
</div>




```python
lineups_df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>country_id</th>
      <th>country_name</th>
      <th>jersey_number</th>
      <th>player_id</th>
      <th>player_name</th>
      <th>team_id</th>
      <th>team_name</th>
      <th>minutes_played</th>
    </tr>
    <tr>
      <th>match_id</th>
      <th>index</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="5" valign="top">7298</th>
      <th>0</th>
      <td>220.0</td>
      <td>Sweden</td>
      <td>16</td>
      <td>4633</td>
      <td>Magdalena Ericsson</td>
      <td>971</td>
      <td>Chelsea LFC</td>
      <td>94</td>
    </tr>
    <tr>
      <th>1</th>
      <td>241.0</td>
      <td>United States of America</td>
      <td>19</td>
      <td>4634</td>
      <td>Crystal Dunn</td>
      <td>971</td>
      <td>Chelsea LFC</td>
      <td>94</td>
    </tr>
    <tr>
      <th>2</th>
      <td>171.0</td>
      <td>Norway</td>
      <td>2</td>
      <td>4636</td>
      <td>Maria Thorisdottir</td>
      <td>971</td>
      <td>Chelsea LFC</td>
      <td>12</td>
    </tr>
    <tr>
      <th>3</th>
      <td>68.0</td>
      <td>England</td>
      <td>24</td>
      <td>4638</td>
      <td>Drew Spence</td>
      <td>971</td>
      <td>Chelsea LFC</td>
      <td>55</td>
    </tr>
    <tr>
      <th>4</th>
      <td>171.0</td>
      <td>Norway</td>
      <td>18</td>
      <td>4639</td>
      <td>Maren Mjelde</td>
      <td>971</td>
      <td>Chelsea LFC</td>
      <td>94</td>
    </tr>
  </tbody>
</table>
</div>
