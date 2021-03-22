# Movie Recommender with Apache Mahout on Amazon Elastic MapReduce (EMR)

## Step 1 : Setup Clusters

* Sign up for an AWS account.
* Start up an EMR cluster (you can choose the cluster in your preference, this experiment is done with one m5.xlarge master and three m5.xlarge slave).

## Step 2: Train the model

* Get the [MovieLens data](http://grouplens.org/datasets/movielens).

```
wget http://files.grouplens.org/datasets/movielens/ml-1m.zip
unzip ml-1m.zip
```
* Convert ratings.dat, trade “::” for “,”, and take only the first three columns:

```
cat ml-1m/ratings.dat | sed 's/::/,/g' | cut -f1-3 -d, > ratings.csv
```
* Put ratings file into HDFS:
```
hadoop fs -put ratings.csv /ratings.csv
```
* Run the recommender job:
```
mahout recommenditembased --input /ratings.csv --output recommendations --numRecommendations 10 --outputPathForSimilarityMatrix similarity-matrix --similarityClassname SIMILARITY_COSINE
```
 * Look for the results in the part-files containing the recommendations:
```
hadoop fs -ls recommendations

hadoop fs -cat recommendations/part-r-00000 | head
```
You should see a lookup file that looks something like this (your recommendations will be different since they are all 5.0-valued and we are only picking ten):

User ID	(Movie ID : Recommendation Strength) Tuples

5	[2572:5.0,2968:5.0,3298:5.0,1517:5.0,3035:5.0,1188:5.0,1649:5.0,198:5.0,3100:5.0,265:5.0]

10	[2374:5.0,1449:5.0,2112:5.0,2243:5.0,1320:5.0,3168:5.0,2110:5.0,3035:5.0,2376:5.0,1717:5.0]

15	[3555:4.5492744,3646:4.4911823,1387:4.467978,3484:4.4360905,3755:4.4323297,3654:4.4251847,3565:4.4034004,3744:4.401004,3793:4.400009,3949:4.3978662]

20	[524:5.0,3301:5.0,2707:5.0,3755:5.0,3624:5.0,2841:5.0,3499:5.0,3298:5.0,266:5.0,3617:5.0]

25	[2375:5.0,1449:5.0,1584:5.0,2376:5.0,1320:5.0,1321:5.0,3298:5.0,3034:5.0,529:5.0,3300:5.0]

30	[2144:5.0,1090:5.0,1333:5.0,2109:5.0,3362:5.0,1956:5.0,1321:5.0,1250:5.0,2371:5.0,1960:5.0]

35	[3035:5.0,3629:5.0,3100:5.0,2969:5.0,1188:5.0,2110:5.0,1913:5.0,1253:5.0,2968:5.0,3167:5.0]

40	[2968:5.0,3168:5.0,594:5.0,3035:5.0,3034:5.0,2376:5.0,3628:5.0,1253:5.0,2110:5.0,3296:5.0]

45	[265:5.0,2243:5.0,2111:5.0,3167:5.0,529:5.0,2375:5.0,2110:5.0,1188:5.0,3035:5.0,3168:5.0]

50	[2289:4.466993,785:4.455226,1171:4.453465,551:4.451631,2064:4.45048,2890:4.44947,2912:4.4343204,2959:4.433308,778:4.4282284,2918:4.4263988]

Where the first number is a user id, and the key-value pairs inside the brackets are movie-id:recommendation-strength tuples.

The recommendation strengths are at a hundred percent, or 5.0 in this case, and should work to finesse the results. This probably indicates that there are many more than ten “perfect five” recommendations for most people, so you might  calculate more than the top ten or pull from deeper in the ranking to surface less-popular items.

## Step 3 : Building a Service

* Next, we’ll use this lookup file in a simple web service that returns movie recommendations for any given user.

1. Get Twisted, and Klein and Redis modules for Python.
    ```
    sudo pip3 install twisted
    sudo pip3 install klein
    sudo pip3 install redis
    ```
2. Install Redis and start up the server.
    ```
    wget http://download.redis.io/releases/redis-2.8.7.tar.gz
    tar xzf redis-2.8.7.tar.gz
    cd redis-2.8.7
    make
    ./src/redis-server &
    ```
3. Build a web service that pulls the recommendations into Redis and responds to queries.
Put the following into a file, e.g., “hello.py”

    ```
    from klein import run, route
    import redis
    import os

    # Start up a Redis instance
    r = redis.StrictRedis(host='localhost', port=6379, db=0)

    # Pull out all the recommendations from HDFS
    p = os.popen("hadoop fs -cat recommendations/part*")

    # Load the recommendations into Redis
    for i in p:

    # Split recommendations into key of user id
    # and value of recommendations
    # E.g., 35^I[2067:5.0,17:5.0,1041:5.0,2068:5.0,2087:5.0,
    #       1036:5.0,900:5.0,1:5.0,081:5.0,3135:5.0]$
    k,v = i.split('\t')

    # Put key, value into Redis
    r.set(k,v)

    # Establish an endpoint that takes in user id in the path
    @route('/<string:id>')

    def recs(request, id):
    # Get recommendations for this user
    v = r.get(id)
    return 'The recommendations for user '+id+' are '+v.decode()


    # Make a default endpoint
    @route('/')

    def home(request):
    return 'Please add a user id to the URL, e.g. http://localhost:8085/1234n'

    # Start up a listener on port 8085
    run("localhost", 8085)

    ```

4. Start the web service.
    ```
    twistd -noy hello.py &
    ```
5. Test the web service with user id “34”:
    ```
    curl localhost:8085/34
    ```
    You should see a response like this (again, your recommendations will differ): 
    The recommendations for user 34 are [2112:5.0,2376:5.0,2110:5.0,2111:5.0,265:5.0,529:5.0,3298:5.0,3035:5.0,1188:5.0,3168:5.0]

### Note : When you’re finished, don’t forget to shut down the cluster.
