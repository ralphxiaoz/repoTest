
# Scrapy Use Cases
## Example One

In this example, we are going to use scrapy to scrape top-rated movies from imdb.

### 1. Creating a new Scrapy project
Before you can do anything else, you need to create a new Scrapy project.
Under your chosen directory, run the following code:


```python
scrapy startproject imdbTop
```

### 2. Define/write the spider

The class that does the crawling is called Spider. You write the spider, and Scrapy uses it to scrape information from a website or websites.  
Spiders must subclass scrapy.Spider and define the initial requests to make, optionally how to follow links in the pages, and how to parse the downloaded page content to extract data.


The following is the code for our spider. We are trying to get the movie titles and the ratings for these top-rated movies.  
Save the code in a file named 'ratings_spider.py'. Put this file under the **imdbTop/spiders** directory in your project.


```python
import scrapy


class RatingsSpider(scrapy.Spider):
    name = "ratings"

    
    start_urls = ['http://www.imdb.com/chart/top']
    
    def parse(self, response):
        for row in response.css('tr')[1:-2]:
            yield {
                'title': row.css('a::text').extract()[2],
                'rating': row.css('strong::text').extract_first()
            }

```

**name** Identifies your spider. Keep in mind that you cannot use the same name for multiple spiders in your project.  
**start_urls** Can take a list of urls. A shortcut to the default **start_requests** method.  
**parse** Handles the response downloaded for each of the requests made. 






### 3. Run the spider & extract data

Go to the top level of your project and run the following code:


```python
scrapy crawl ratings
```

Your output will look similar to this:

    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200   http://www.imdb.com/chart/top>  
    {'title': 'The Shawshank Redemption', 'rating': '9.2'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': 'The Godfather', 'rating': '9.2'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': 'The Godfather: Part II', 'rating': '9.0'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': 'The Dark Knight', 'rating': '8.9'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': '12 Angry Men', 'rating': '8.9'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': "Schindler's List", 'rating': '8.9'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': 'Pulp Fiction', 'rating': '8.9'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': 'The Lord of the Rings: The Return of the King', 'rating': '8.9'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': 'The Good, the Bad and the Ugly', 'rating': '8.8'}  
    2017-02-18 19:45:39 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>  
    {'title': 'Fight Club', 'rating': '8.8'}  

### 4. Store scraped data

To store scraped data, just run the following command.


```python
scrapy crawl ratings -o ratings.json
```

Now you have a .json file that contains the scraped movie ratings data.  
It is important to note that Scrapy appends to a given file instead of overwriting its contents, so if you run this command twice without removing the file before the second time, you might end up with a broken json file.

You can also store the scraped data in other formats such as JSON Lines.


```python
scrapy crawl ratings -o ratings.jl
```

## Example Two

In this example, we are going to use item pipeline to filter scraped movies whose production year is after year 2000. We'll continue to use the code from imdbTop example.

To define common output data format Scrapy provides the Item class. Item objects are simple containers used to collect the scraped data. They provide a dictionary-like API with a convenient syntax for declaring their available fields.

### 1. Declaring Items

First we declare movie items in **imdbTop/itmes.py** using following code:


```python
import scrapy

class Movie(scrapy.Item):
    # define the fields for your item here like:
    title = scrapy.Field()
    rating = scrapy.Field()
    ranking = scrapy.Field()
    year = scrapy.Field()
    pass
```

Then we add itmes to spider "ratings_spider.py". Here we added some new data to scrape, like raking and year.


```python
import scrapy
from imdbTop.items import Movie

class RatingsSpider(scrapy.Spider):
    name = "ratings"

    start_urls = ['http://www.imdb.com/chart/top']
    
    def parse(self, response):
        movies = []
        for row in response.css('tr')[1:-2]:
            movie = Movie()
            movie['title'] = row.css('a::text').extract()[2]
            movie['rating'] = row.css('strong::text').extract_first()
            movie['ranking'] = row.css('td.titleColumn::text').extract()[0].replace('\n', '').replace(' ','').replace('.','')
            movie['year'] = row.css('span.secondaryInfo::text').extract()[0][1:-1]
            movies.append(movie)
        return movies

```

After an item has been scraped by a spider, it is sent to the Item Pipeline which processes it through several components that are executed sequentially.

We can define a pipeline in **imdbTop/pipelines.py**:


```python
from scrapy.exceptions import DropItem

class DropMoviePipeline(object):

    def process_item(self, movie, spider):
        if int(movie['year']) > 2000:
            return movie
        else:
            raise DropItem("Movie \' %s \' is before year 2000" % movie['title'])
```

In the pipeline defined above, we are returning the movies whose product year was larger than 2000.

Finally, to activate the pipeline, to activate an Item Pipeline component you must add its class to the ITEM_PIPELINES setting in **imdbTop/settings.py**:


```python
ITEM_PIPELINES = {
   'imdbTop.pipelines.DropMoviePipeline': 300,
}
```

We have implemented all the code needed to filter the movies. Go to the top level of your project and run the following code:


```python
scrapy crawl ratings
```

you should see something like this in your console:

    2017-02-19 19:56:42 [scrapy.core.scraper] WARNING: Dropped: Movie ' The Shawshank Redemption ' is before year 2000
    {'ranking': '1',
     'rating': '9.2',
     'title': 'The Shawshank Redemption',
     'year': '1994'}
    2017-02-19 19:56:42 [scrapy.core.scraper] WARNING: Dropped: Movie ' The Godfather ' is before year 2000
    {'ranking': '2', 'rating': '9.2', 'title': 'The Godfather', 'year': '1972'}
    2017-02-19 19:56:42 [scrapy.core.scraper] WARNING: Dropped: Movie ' The Godfather: Part II ' is before year 2000
    {'ranking': '3',
     'rating': '9.0',
     'title': 'The Godfather: Part II',
     'year': '1974'}
    2017-02-19 19:56:42 [scrapy.core.scraper] DEBUG: Scraped from <200 http://www.imdb.com/chart/top>
    {'ranking': '4', 'rating': '8.9', 'title': 'The Dark Knight', 'year': '2008'}


Also, you could use following code to store results in JSON file:


```python
scrapy crawl ratings -o movies_after_2000.json
```
