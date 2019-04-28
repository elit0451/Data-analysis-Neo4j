# Data analysis with Neo4j <img src="https://go.neo4j.com/rs/710-RRC-335/images/neo4j_logo_globe.png" height="40" align="center">

The data we are going to analyse is Twitter data <img src="https://seeklogo.com/images/T/twitter-2012-positive-logo-916EDF1309-seeklogo.com.png" height="15" align="center"> from the British Islands <img src="https://cdn.pixabay.com/photo/2012/04/18/01/46/united-36481_960_720.png" height="30"> found under [this link](http://followthehashtag.com/datasets/170000-uk-geolocated-tweets-free-twitter-dataset/)<img src="https://bonnicilawgroup.com/wp-content/uploads/2015/08/map-marker-icon.png" height="20">.

To import the data into Neo4j we will use a [CSV file](./some2016UKgeotweets.csv.zip) <img src="https://cdn4.iconfinder.com/data/icons/file-types-5/96/document_file_extension_format_type_csv-512.png" height="20" align="center">

<br/>

----
## Excercise 1 - Cypher for loading the tweets <img src="https://www.iconsdb.com/icons/preview/royal-blue/code-xxl.png" width="50" align="center">

- To spin a docker :whale: container the following command will be necessary: </br>
`docker run -d --rm --name neo4j --publish=7474:7474 --publish=7687:7687 -v $(pwd):/var/lib/neo4j/import --env NEO4J_AUTH=neo4j/fancy99Doorknob --env NEO4J_dbms_memory_heap_max__size=1024M neo4j`

_NB_:exclamation:If you are a Windows user <img src="https://images.all-free-download.com/images/graphiclarge/windows_81_default_icon_pack_6830210.jpg" height="20" align="center"> , please replace **$(pwd)** with the path where the CSV file is kept. </br>
&nbsp;&nbsp;&nbsp;&nbsp; :exclamation:Also pay attention to the command `NEO4J_dbms_memory_heap_max__size=1024M` which would make sure the container <img src="https://images.all-free-download.com/images/graphiclarge/freight_container_design_vector_587738.jpg" height="20" align="center"> has enough memory for the entirety of the data.

</br>

A **Cypher command** which loads the tweets, creates a number of nodes labeled **Tweet** with the properties _username_, _nickname_, _bioplace_, _latt_, _long_ and _content_ (corresponding to the columns "User Name, "Nickname", "Place (as appears on Bio)", "Latitude", "Longitude" and "Tweet content" in the CSV file), and a list of mentions to each Tweet - _mentions_:
</br>

```javascript
LOAD CSV WITH HEADERS FROM "file:///some2016UKgeotweets.csv" AS row 
    FIELDTERMINATOR ";"
WITH row["Tweet content"] AS tweetContent, row
WITH extract( m in 
                filter(m in split(tweetContent," ") WHERE m STARTS WITH "@" AND size(m) > 1) 
                | right(m,size(m)-1)) AS mentionedusr, row
WHERE NOT row["Tweet Id"] IS NULL AND NOT toFloat(row["Latitude"]) IS NULL AND NOT toFloat(row["Longitude"]) IS NULL
CREATE (tw:Tweet {
    username:row["User Name"],
    nickname: row["Nickname"],
    bioplace: row["Place (as appears on Bio)"],
    latt: toFloat(row["Latitude"]), 
    long: toFloat(row["Longitude"]),
    content: row["Tweet content"],
    mentions: mentionedusr
    })
```
</br>

> The output of the query is: </br> **Added 136087 labels, created 136087 nodes, set 952609 properties, completed after 9980 ms.**

<br/>

----
## Excercise 2 - New set of nodes and relations <img src="https://s3.amazonaws.com/dev.assets.neo4j.com/wp-content/uploads/20161101022535/node-js-react-js-neo4j-template.png" width="50" align="center">

A **Cypher command** which creates a new set of nodes labeled **Tweeters**, whith a _MENTIONS_ relation:
</br>

```javascript
MATCH(tweet:Tweet)
UNWIND tweet.mentions AS mentions
WITH mentions, tweet
WHERE mentions <> ""
CREATE (user:Tweeter {name:mentions})
CREATE (tweet)-[:MENTIONS]->(user)
```
</br>

> The output of the query is: </br> **Added 41030 labels, created 41030 nodes, set 41030 properties, created 41030 relationships, completed after 2149 ms.**

Part of the graph :arrow_lower_right:
<p align="center">
<img src="https://waffleio-direct-uploads-production.s3.amazonaws.com/uploads/5b631124103d580013dcf6a4/125516c66e82c728ace21e0d46db978826878dba87e6ab03f60da1cd6416733e7b5ce37a27cbb17cf1172c43434d0eee1e020a17b8eb8339a3e46979820e5ae8d76b5d72943411adb0d41beb57bb72895d99a4fedb1294b7607caddecd5d340c071883fe5830a69d4dcd59f348e94b54b65e.png" width="60%">
</p>

</br>
</br>

A **Cypher command** which creates a relation _TWEETED_ between **Tweeters** and **Tweet**:
</br>

```javascript
MATCH(tw:Tweeter)
WITH tw
MATCH(tweet:Tweet {username: tw.name})
CREATE (tw)-[:TWEETED]->(tweet)
```
</br>

> The output of the query is: </br> **Created 162 relationships, completed after 778 ms.**

Part of the graph :arrow_lower_right:
<p align="center">
<img src="https://waffleio-direct-uploads-production.s3.amazonaws.com/uploads/5b631124103d580013dcf6a4/125516c66e82c728ace21e0d46db978826878dba87e6ab03f60da1cd6416733e7b5ce37a27cbb67ff1122e43434d0eee1e020a17b8eb8339a3e46979820e5ae8d76b5d72943411adb0d41beb57bb72895d99a4fedb1294b7607caddecd5d340c041083fe5830a69d4dcf58fe4be54e53b65e.png" width="60%">
</p>

<br/>

----
## Excercise 3 - Spatial analysis <img src="https://geodiscover.alberta.ca/geoportal/catalog/images/icon-layers-pin.png" height="45" align="center">

A **Cypher command** which creates new nodes **Distance** and much more:
</br>

```javascript
MATCH (tw:Tweeter)-[:TWEETED]-(tweet:Tweet)
WITH DISTINCT tweet as distTweet
WITH Collect(DISTINCT point({ longitude: distTweet.long, latitude: distTweet.latt })) as Points, distTweet.username as DistUser
CREATE (distNode:Distance {name:DistUser, maxDistance: 0})
FOREACH (p1 IN Points |
	FOREACH (p2 IN Points |
		FOREACH (greater in CASE
			WHEN round(distance(p1, p2)) > distNode.maxDistance THEN [1]
			ELSE [] END | SET distNode.maxDistance = round(distance(p1, p2)))
))
```

All the points belonging to all the tweets of a user are added to a list and after that new node with label **:Distance** is created to hold the value of the maximum distance between that user's tweets, initializing it with 0. We then compare all the points in the list with each other and when a value higher than the one we currently store we update _maxDist_.

</br>

A **Cypher command** which finds the top 10 list of **Tweeters** whose **Tweets** are the furtherst apart:
</br>

```javascript
MATCH (d:Distance)
WITH d.maxDistance / 1000 AS distance, d.name AS username
ORDER BY distance DESC
RETURN DISTINCT username, distance
LIMIT 10
```
</br>

> RESULTS

| Username | Furthest Distance (km)|
| :----:|:----:|
|	holly |459.592|
|	scattaranks|313.259|
|	Zara|219.887| 
|	SparkyFree|21.058| 
|	c0axial|20.778|
|	Coral|19.254|
|	jorgitocorchea |10.577|
|	huytonbad|7.423|
|	mario|4.336| 
|	urbanvox|4.158| 


<br/>

___
> #### Assignment made by:   
`David Alves 👨🏻‍💻 ` :octocat: [Github](https://github.com/davi7725) <br />
`Elitsa Marinovska 👩🏻‍💻 ` :octocat: [Github](https://github.com/elit0451) <br />
> Attending "Databses for Developers" course of Software Development bachelor's degree
