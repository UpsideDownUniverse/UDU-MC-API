# UDU Minecraft Server API
#### The planning and study phase.

Okay in this documentation, we will be discussing the  
steps involved in the creation of this UDU MC API, discussing  
what it is, it's functionality, how it works, and many such things,  

wew this is gonna be a long Markdown file,  
*breathes in*

- - -

### What is this Minecraft Server API? What is it's use?

This minecraft server API will be a simple server side Rest API, which will be posting the data of the players on the server, which will include data such as Health, armor, enchants, achievements (⚠️), etc...  

In the creation of this API, the only major road-block will be the problem of attempting to request the data of an offline player, but we have a fix to that brainstormed and ready for application.

The only use-case of this API is for using/getting the data of a player and displaying it on the UDU MC Website, other use-cases can be to use the same data in the Minecraft-discord bot (NOTE: Not like the DiscordSRV bot!), or to use the data for something fun... 👀

### What is the development Timeframe and cost?

Lets say we begin on `13th of June, 2022` and assign 2 weeks for development and 2 weeks for testing we should be done within a month.

When it comes to cost...   
I do not think it will cost anything as we are not using any external help.

### How will we make this API?

Read below, this section of the README.md is only to answer simple questions...

- - -

# API Creation

- - -

## Method 1 (⚠️ Only `uuid`, `timeplayed` & `username`):
In this method, we can only fetch data such as the UUID, time-played and username using the stats file on the Minecraft server, as minecraft servers now store each player's data in the `./world/stats` folder we will be using python to fetch this data and put it onto a MySQL server and send that data to be rendered onto a PHP server. (Note: This is not the final method of usage!)

```
NOTE: This method is not viable as we are looking to fetch more than just UUID, time-played, and username! 
But as we are in the development phase, let us understand how this works.
```

### prerequisites
1. A Minecraft server

We will need a minecraft server of course because this is a minecraft plugin afterall, we will be using the `1.18.2` version of spigot for development as spigot is the base architecture among almost all vanilla minecraft server distros. (NOTE: UDU MC NETWORK runs on papermc version `1.18.2`) we are using spigot as it will be easier to migrate to paper from here.

2. Database server

We will be using a MySQL database server to store data which is fetched from the minecraft server (NOTE: This is the fix that has been discussed for the offline players data problem). We are using MySQL because it is the most common and easily operable database server, but we can switch to GraphQL or MongoDB later on as it will be easier to provide `json` data directly.

3. Python (Language)

Yes, python, we will be using python to facilitate communication between the Minecraft server and the Web server, this will be mostly done using websockets and also we will be adding encryption logics to encrypt data flowing between the Minecraft server and the Web server, we are using python because it is light weight and fast, and comes bundled with most of the Linux distributions, and as we are using the LAMP stack on the web server i.e, `L`inux `A`pache `M`ySQL `P`HP tech stack, it will be simpler on the webserver side. (NOTE: 🤔💭 Implementation of MERN stack instead...)

4. Crontab

Cron jobs will be used to trigger timed fetching of data from the minecraft server, this will be rate-limited to around 1 request per 5 seconds (To reduce stress on the server and free up bandwidth), crontab is also pre-bundled with most of the linux distros. Fetching will either happen based on time intervals and/or triggered by various events on either the Minecraft server or the Webserver.

5. Webserver

We will be using `Apache` for this project, but it doesn't really matter which webserver we are using as we are just running the data out for a website to fetch externally in a different or same machine. Apache is fast and offers a lot of features and is easy to setup so we will be using it in this project.

### Method of execution

Lets start from the user-end, when a user visits `www.upsidedownuniverse.xyz/{profile}` the user sends a request to the PHP server, the PHP server grabs the data from the MySQL Database, and returns it to the PHP page that is being served. We can also get the player's avatar / skin from an external API `https://mc-heads.net/` by passing the request to `/body/{username}` endpoint. For this data to be the latest, the cronjob will get the data and post it onto the MYSQL server.

Below is a flowchart diagram to see how we will be doing the method of execution explained above.

![Method 1 flow chart](https://github.com/UpsideDownUniverse/UDU-MC-API/blob/1e82e7b082c328e9294a457b6d7efc164b5ab6c3/assets/UDU%20MC%20API%20method%201%20(1).svg)

Python MC -> DB file code (alpha v0.1.0):  
This code will collect data from the server files and upload it to the Database.

connector.py

```python
import os
import mysql.connector as sql
import json

path="{path_to_stats_file}"
directories = os.scandir ( path )
uuid=""

db = sql.connect(
  host="mysql_db_server",
  user="connection_user",
  passwd="password",
  database="exposed_database"
)

cursor = db.cursor()

for entry in directories():
  if entry.is_file():
    uuid = os.path.splittext(entry.name)[0]
    print(entry.name)
    file = open(path + '/' + entry.name)
    fileparsed = json.load(file)
    play_time = fileparsed["stats"]["minecraft:custom"]["minecraft:play_time"]
    query = "SELECT COUNT(*) FROM players WHERE uuid = '"+uuid+"'"
    cursor.execute(query)
    player_count=cursor.fetchone()
    print(player_count[0])
    if player_count == 0:
      query = "INSERT INTO players (uuid, play_time) VALUES ('"+uuid+"', '"+str(play_time)+"');"
    else:
      query = "UPDATE players SET play_time = '"+str(play_time)+"' WHERE uuid = '"+uuid+"'"
    print(query)
    cursor.execute(query)
    db.commit()
```

Database example view:    
![Database screenshot](https://github.com/UpsideDownUniverse/UDU-MC-API/blob/b81b9385ffe85432a6a3a7dadf2a7c7e92deea7c/assets/Screenshot%202022-05-29%20172857.png)

PHP DB -> Webserver connection code (alpha v0.1.0):
This code will take the data in the database and render it onto a site.

```php
<?php
require('udu-mc-apy.php');

function ConvertToHoursMins($time, $format = '%02d:%02d') {
  if($time < 1){
    return;
  }
  $hours = floor($time / 60);
  $minutes = ($time % 60);
  return sprintf($format, $hours, $minutes);
}
    
    define( 'DB_NAME', 'exposed_database' );
    define( 'DB_USER', 'connection_user' );
    define( 'DB_PASSWORD', 'password' );
    define( 'DB_HOST', 'exposed_db_server' );
    
    $conn = mysqli_connect( DB_HOST, DB_USER, DB_PASSWORD, DB_NAME );
    if (!conn) {
      die("Connection to database FAILED! reason: " . mysqli_connect_error());
    }
    $sql = "SELECT * FROM players ORDER BY play_time DESC";
    
    if ($result = mysqli_query($conn, $sql)) {
      echo(" {SOME BORING HTML SCRIPT HERE} ");
      while ($row = mysqli_fetch_row($result)) {
        if (strlen($row['0']) > 0) {
          $play_time_hours = (int)$row['2'] /20 /60;
          
          echo(" {BORING HTML STUFF AGAIN} ") //NOTE: over here if we use .$row['0'] it is basically the uuid of the player.
        }
      }
    }
?>
```

That is the end of Implementation method 1 

Some of its Pros and Cons:

| PROS          | CONS           |
|:-------------:|:--------------:|
| Simple & Fast      | Needs to run on MC server machine |
| Can grab required data      | Data selection limited to 3 variables      |

`NOTE: We can grab additional data, but it is not viable, as the problem rests where the data is limited to what the Minecraft stats.json file provides`

- - -

## Method 2 (from `spigot & bukkit`) 🚀:

In this method we will be using the java method of approaching the problem, by using the bukkit and spigot libries to fetch data on the server and upload it to a MySQL database, this data will then be fetched by an `RESTful API server` which will convert the schema into usable json files on rest urls which can be later accessed by the frontend using react API hooks, or something of that manner.

```
Note: this method will involve creating a java plugin for the minecraft server that will of course run on the minecraft server.
```

### Prerequisites

1. A Minecraft Server

As said in `Method 1` .

We will need a minecraft server of course because this is a minecraft plugin afterall, we will be using the `1.18.2` version of spigot for development as spigot is the base architecture among almost all vanilla minecraft server distros. (NOTE: UDU MC NETWORK runs on papermc version `1.18.2`) we are using spigot as it will be easier to migrate to paper from here.

2. Database server

As said in `Method 1` .  

We will be using a MySQL database server to store data which is fetched from the minecraft server. We are using MySQL because it is the most common and easily operable database server, but we can switch to GraphQL or MongoDB later on as it will be easier to provide `json` data directly.

We have considered using a MySQL x MongoDB architecture here to eliminate the need for a RESTful API server, but that option is only in theory not practically done.

3. Webserver

As said in `Method 1` .

We will be using `Apache` for this project, but it doesn't really matter which webserver we are using as we are just running the data out for a website to fetch externally in a different or same machine. Apache is fast and offers a lot of features and is easy to setup so we will be using it in this project.

### That is all, actually, this method requires lesser components and gets things done in a simpler faster way.

- - - 

### Method of execution

In this method, we will have to make a java plugin to run on the server, the jar plugin will be light-weight because it does not run too many operations on the server.

This plugin should have the following funtions:

- [x] Ability to run on express.js
- [x] Ability to get player name, UUID, stats, health, experience, level, hunger, deaths, mob kills, etc...
- [x] Ability to upload that data onto a database

After the data has been uploaded onto the database server, the RESTful API app should grab the data from the MySQL server and initialize the API endpoints, we can also do this in a different method, we can let the user, i.e, the browser send an API request to the RESTFUL API server and then the server fetches data from the database.

Below is a flowchart diagram to see how we will be doing the method of execution explained above.

![Method 2 flowchart](https://raw.githubusercontent.com/UpsideDownUniverse/UDU-MC-API/1387c9a1c5397969369806683f773970a1931778/assets/UDU-MC-API-Method%202.drawio.svg)

Java plugin code -> UDU Minecraft Server (alpha v0.2.0):  
This is the plugin that will go on the minecraft server.

Java plugin development is still underway so i will mention only the code that will fetch the required data mentioned in the method's explanation.

```java
// Player username
"username", player.getName()

// Player UUID
"uuid", player.getUniqueId().toString()

// Player Health
"health", player.getHealth()

// Player Food
"food", player.getFoodLevel()

// Player World
"world", player.getWorld().getName()

// Player Experience
"experience", player.getExp()

// Player Level
"level", player.getLevel()

// Player Deaths
"deaths", player.getStatistic(org.bukkit.Statistic.DEATHS)

// Player Kills
"kills", player.getStatistic(Statistic.MOB_KILLS)

// Player Jumped
"jumps", player.getStatistic(Statistic.JUMP)
```

The above are the variables that we will be pulling from the server.

MySQL database (alpha v0.2.0):  
No data collected on this topic just yet, because this method is still under trial and error, we do not have an exact roadmap to follow on this one.  

That is the end of Implementation method 2  


Some of its Pros and Cons:

| PROS          | CONS           |
|:-------------:|:--------------:|
| Fastest & Server-side      | Need to run java plugin on Minecraft server. |
| Can grab any type of available data.      | Limited to Spigot available data.      |


`NOTE: This is the best method of approach for this project, there might be other methods, but those are still under research.`

- - -
