Problem

C-Movie wants to create a movie rating aggregation platform. It plans to source ratings from a variety of partner platforms (e.g. IMDb, Rotten Tomatoes, and similar) and, based on them, to calculate their market-unique C-Rating™.


C-Movie gets ratings from partner platforms in a variety of ways: via REST APIs, online or offline data feeds generated periodically (e.g. a CSV file left on an FTP server).


Each platform provides:

    Rating. Ratings are expressed in various grading systems (e.g. 0-10, 0-100%). Some of the platforms additionally provide ratings split by different categories (not standardised between platforms). 
    Movie title
    Genre(s). A movie can be described with multiple genres. Genres are not standardised between platforms and may not match between providers (e.g. one can return Drama, Documentary where another one can describe the same movie as Drama, Mystery).


C-Movie ratings are going to be available as a web application and through a REST API. In the MVP, C-Movie wants to provide at least movie search returning:

    Movie title. The movie title is taken from a platform that C-Movie considers the most reliable (the used provider may vary for each movie).
    Genre. The movie genre may be taken from multiple platforms that C-Movie considers most reliable (as with the movie title, providers used as the source for the genre may be different for each movie).
    C-Rating, a normalised total movie rating between 0 and 100 split into 3 categories: performance, screenplay, soundtrack.


C-Rating is calculated using a sophisticated proprietary algorithm taking into account provider total ratings and ratings by category.


Solution

Provide a solution design of the C-Movie business problem: 

    As a textual description (e.g. by providing a link to a Github repository or a Google Doc). Document any assumptions or constraints you made during the process: both business and technical. 
    Limit your description to max. 1-2 page excluding diagrams. 
    ﻿No code is required.
    You may include any supporting materials (diagrams, drawings, etc.) that may help convey your ideas.


﻿The solution you send us will be the starting point for a conversation with our tech team about the design and decisions you made (assumptions, constraints, trade-offs, non-functional requirements).