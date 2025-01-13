
# Movie-C Solution


**MovieC** as an aggregator platform that calculates an aggregate version of movies from different data sources. Is using web with event driven capabilities architecture. 

## Assumptions

The system is desigined based in the following assumptions:

- A movie is update regularly by each data source. 
- The movies contains a unique id that is shared through all data sources. (movie_id)
- MovieC works as a broker for information for another websites.

## Architecture

The architecture is divided in 3 systems:

- [web system](#architecture-web)
- [extractor](#architecture-extractor)
- [aggregator](#architecture-agregator)

### Architecture Web

I choose to use a web architecture because allow to centralize business logic in one single executor, it is easy to make adjustments and improvements and also centralizes the monitoring of the system.

Also one key advantage is that the core aggregation mechanism is not part of the web system making it more robust to failure from the aggregator system

```plantuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml


System_Boundary(c1, "MovieC Core System") {
    Container(website, "Movie-C Website", "Movie-C")
    Container(api, "MovieC API", "Backend API for accesing RE Database")
    ContainerDb(db, "Database Movie-C", "RE", "Movie Aggregated information")
    ContainerDb(bucket, "Data Assets", "files", "Movie Assets")
    Rel(api, db, "sql", "request movie information")
    Rel(api, bucket, "s3", "request movie asset")
}
```

- **Database**: Saved all the data need by the system to stored the aggregated information from all the data sources
- **Bucket**: Store assets like images, or small videos or trailers if is need it. 
- **API**: it contains the backend and the apis that support the web page. 
- **Website**: contains all the assets for web distribution through a web browser. 

### Architecture Extractor

The Extractor gets the data from data sources and transform in entites that can be used for aggregation.

For creating such process we need to introduce certain concepts that allow us to ensure the quality of the data:

- **Entity Uniqueness**: How to ensure that one movie is the same movie in all data sources? Define a way to identified the movie in a unique way? is there an id that can be used for all the data sources? The system needs to define an algorithm that can be used for all the data sources to be able to match them (is this movie A the same movie A in this data source?)

```python
def get_movie_id(args)
    # return a movie id. 
    return args["id"]
```
A `movie id` is an id that can be used for matching movies in all data sources. 

Note: This is going to cover the majority of the movies, but there is always going to be some movies that we can not match between data sources, Is suggested to have a log table where we can check what movies are we not able to get the id, and what is the porcentage in the overall data source. 

*For continue the system i assume that all the titles from the original country are the same for all data sources and the dates of registration are unique*

- **Metadata**: What data is relevant and how the metadata can be override. Define rules for each field that defines a movie, also consider scenarios what happened if the data rewrites information as empty or not data should we also saved as empty?

We can define the same blueprint for all data sources pipeline:

```plantuml
@startuml Movie C
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

System_Boundary(c2, "Data Source") {
    Container(source, "Data Source X")
}

System_Boundary(c1, "Movie Data Extractor") {
    Container(engine, "Extractor for Source X", "Only is executed when there is a changed in the original data source")
    ContainerDb(db_aggregator, "Data Storage", "Bucket", "Store the metadata information from a data source")
   
    Rel(engine, db_aggregator, "file", "Check if there are changes or is new")

}

ContainerDb(movie_update_events, "MQ", "Movie Update Events")

Rel(engine, movie_update_events, "MS", "Push there is an update for movie('A')")
Rel(c2, c1, "Get Updates from Source (Event)")
@enduml
```

The source updates can be trigger by an SFTP server or by an scheduler. This can be implemented using technologies like S3 where we can trigger a serverless function when an update happened in one file in the bucket.

Note: For big updates like the first runs, (big csv files for example) in a data sources this can be a task that is not able to be run in a normal container or a serveless function because of the memory restrictions, is suggested to used technologies like spark that can be run serverless in AWS using glue jobs. 

*All data sources needs to follow the same schema*, This allow the system to compare data sources that has similar data and also define what information is strictly important for been able to use it in the aggregation algorithm. This schema is going to define what is a movie in the system:

```plantuml
    class Movie {
        source : Where is this information coming (IMDB, Rotten Tomatoes)
        source_id: unique id of the movie for data source
        title : Original title.
        country_of_origin: Country of where the movie was made
        source_rating[] : All the ratings that has the data source for movie
        registration_date: When was the movie registered in the movie database
        names[] : All the possible names that movie has in different lenguajes
        genres[] : Movie genres
        crating_performance: C-Rating for Performance  
        crating_soundtrack: C-Rating for Soundtrack
        crating_screenplay: C-Rating for ScreenPlay
        ..another metadata
    }
```

- Each data sources saves the movie information in the `Aggregated Storage`. 
- Each time there is a **changed** in the data the system is notified through an event `notified_movie_has_updates('movie_id')`. 

### Architecture Agregator

The aggregator is the main part of the sytem that calculates the final scored of a movie. It contains the aggregator algorithm that is going to be used for calculating the final score. 

```plantuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

System_Boundary(c3, "Movie Data Aggregator") {
    Container(aggregator_engine, "Aggregator", "Only is executed when there is a changed in the original data source")
    ContainerDb(movie_update_events, "MQ", "Movie Update Events")
    Rel(movie_update_events, aggregator_engine, "Get updates for movies")
}
```

The aggregator algorithm containes all the rules that are define for each data source and each field to populate a movie entity and stored in db.

```python
    def get_info_for_movie(movie_id)
        # Get movie info from data sources
        return (imdb_movie, rt_movies ...)

    def calculate_movie_agg(imdb_movie, rt_movies ...)
        # Calculcuate new movie from all data sources
        return MovieC(
            title=get_title(imdb_movie, rt_movies..),
            rating_soundtrack=get_rating_for_soundtracke(imdb_movie, rt_movies...
            ...
        )
    calculate_movie_agg(**get_info_for_movie(movie_id))
```


## Sequence of an Update

Here we can see how an update of a source goes through all the systems:

```plantuml

data_source -> extractor_for_source : There is an update from data source x
extractor_for_source -> extractor_for_source : Iterate through all the information from source x and created Movie(entities)
extractor_for_source -> extractor_for_source : Validate Movie Entity (for empty information)
extractor_for_source -> aggregator_storage: Storage Movie Entities.
aggregator_storage -> extractor_for_source
extractor_for_source --> movie_aggregator:  Notified that there is a changed in source x for single movie.
movie_aggregator -> aggregator_storage: Get the data for movie from all the data sources stored in storage
aggregator_storage -> movie_aggregator
alt for each field 
    movie_aggregator -> movie_aggregator: execute rule for filling field
end

movie_aggregator -> movie_aggregator: Calculate aggregated scored

movie_aggregator -> movie_database: Stored (update, created) movie in database.
```

## Notes in Implementation

The system can be implemented using airflow as a main engine for running extraction But for runing the aggregation is suggested to used a message queue that ensure the order of the updates. This allow to be a more robust system where we can try and re-try operations without putting in risk consistency of the data. This event can be listen by a serverless function that can run the aggregated algorithm and saved the result in database. 



