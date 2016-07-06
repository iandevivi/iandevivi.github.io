---
layout: post
title:  "Scraping for the CLI Data Gem Project"
date:   2016-07-06 18:42:07 +0000
---

### Tougher than it looked

For my CLI Data Gem Project, I searched high and low for a good website that would provide some interesting / useful information and wouldn't be too difficult to scrape.  Some sites were of personal interest to me, but the data I was after wasn't easily attainable due to inconsistent data reporting - one event may have results as a PDF, another might be an HTML or JS.  Other sites would have been great to scrape, but the data were not complete, or didn't have that second layer of data.  

The point of all of this is that I spent **ENTIRELY** too long trying to find the perfect site to scrape and should have just made a damn decision and moved on.  Because after all of my searching for a great site that I would want to scrape and would have sufficient layers and an interesting topic... it turns out I'm still learning and I picked a site that proved to have a hidden challenge.  

I enjoy cooking.  The ability to make delicous (sometimes even healthy) food at home is a rewarding skill to have.  Like anything, the more you do it, the better you get.  I like trying new recipes, and I eventually found this [Food Network Daily Recipes List](http://www.foodnetwork.com/recipes/photos/recipe-of-the-day-what-to-cook-now.html) and decided it would be fun to make a gem out of a list of recipes, allow the user to chose a recipe to learn more about and be able to access ingredients list and the cooking instructions for each recipe.  I also wanted the users to make informed decisions about what recipes to use, and utilizing the cooking time and user ratings was a priority.  Do you really want to put effort into a recipe that is only rated 3 / 5 stars?  

Identifying what to scrape started out pretty straightforward.  Everything I needed was in the unordered list with class "feed" - I just had to drill down from there to get the :name ("h6 a").text, the :cook_time ("dd").text, and the :url (".community-rating-stars").attribute("href").value).  

A weird thing happened when I went about trying to scrape the :rating though.  I couldn't see it in my scraped data. 

```
<section class="review-rating section">
  ::before
  <a class="community-rating-stars" href="/recipes/food-network-kitchens/herb-roasted-chicken-with-melted-tomatoes-recipe#communityReviews">
    <div class="gig-rating-stars" title="4.5 of 5 stars">
    </div>
```

I tried every selector combo I could, but never saw the title text of "4.5 of 5 stars".  It's right there but I can't make it show up!  So off to Slack with the problem, because at this point I'm convinced this was a PICNIC situation.

![PICNIC](https://image.spreadshirtmedia.net/image-server/v1/designs/5445413,width=178,height=178,version=1385034145/PICNIC---Problem-in-Chair,-not-in-Computer---geek---noob---newbie---admin.png) 

After some investigation of the scraped data where the rating info should be, it was apparent that something else was happening here.  Prying into the scraped data and trying out some different selectors that should result in the rating info, we ended up with the following:

```
[3] pry(Scraper)> featured_gallery.css("a.community-rating-stars")[0]
    => #(Element:0x3fcce24114e8 {
      name = "a",
      attributes = [
        #(Attr:0x3fcce2411448 { name = "class", value = "community-rating-stars" }),
        #(Attr:0x3fcce2411434 {
          name = "data-rating",
          value = "{\"id\":\"ef830b06-0438-41ef-a428-367e546a1cee\"}"
          }),
        #(Attr:0x3fcce2411420 {
          name = "href",
          value = "/recipes/food-network-kitchens/herb-roasted-chicken-with-melted-tomatoes-recipe#communityReviews"
          })]
      })
```

No mention of gig-rating-stars anywhere.  But there's a value called "id" with one heck of a string attached to it.  The theory we come up with is that perhaps there's something to this 'gig' nomenclature.  The always helpful [@beingy](https://learn.co/beingy) decides it must be an API source, and some detective work brings him to the conclusion that it's a Gigya id.  Progress!  So how to access the information?  More scouring of the site with the dev tools, leads to the eureka moment.  Buried in the sources tab is a file named 'comments.us1.gigya.com' which provides the following: 

```
gigya._.apiAdapters.web.callback({
   "streamTitle": "herb-roasted chicken with melted tomatoes",
     "streamTags": [],
     "categoryID": "recipe",
     "createDate": 1423893592031,
     "commentCount": 16,
     "approvedCommentCount": 16,
     "threadCount": 16,
     "ratingCount": 16,
     "rssURL": "http://comments.us1.gigya.com/comments/rss/6714431/recipe/ef830b06-0438-41ef-a428-           367e546a1cee",
     "avgRatings": {
       "_overall": 4.5
     },
     "lastCommentTimestamp": 1459628756280,
     "isUserSubscribed": false,
     "moderationMode": "inherit",
     "moderationModes": {
       "text": "inherit",
       "image": "inherit",
       "video": "inherit",
       "url": "inherit",
       "other": "inherit"
     },
     "ratingDetails": {
       "_overall": {
         "ratings": [
           1,
           0,
           0,
           4,
           11
         ],
         "avg": 4.5
       }
     }
   },
   "statusCode": 200,
   "errorCode": 0,
   "statusReason": "OK",
   "callId": "3a0cd170bef44606a928a6a56ab615ce",
   "time": "2016-07-05T21:40:39.155Z",
   "context": "R4190249157"
 });
```

Whoa!  The value ID from the scrape of the recipe website is the same as the rssURL and there's the 
``` 'ratingDetails': "avg": 4.5 ``` that I need!  We're not quite out of the woods yet, however, because this is JSON, and that's going to be a bit different.  Thankfully, StackOverflow to the rescue. There's a [JSON](http://stackoverflow.com/questions/5410682/parsing-a-json-string-in-ruby) gem that allows for easy parsing.  A new method for scraping the JSON code provides the api info that contains the ratings and then using JSON.parse, we're getting a hash that we can inspect and then grab the data we want.  In this instance we just need 
``` [:streamInfo][:avgRatings][:_overall] ```,  ***which finally returns the rating value of 4.5!***  

![Keanu approves](http://media.giphy.com/media/j5QcmXoFWl4Q0/giphy.gif)
Hell yea!  Now it's just a matter of grabbing the api_id in the original scrape method, calling and passing the api_id into this JSON parsing method, which in turn returns the rating and stores it in recipe.rating!

This was challenging, but it was a fun challenge of going down the rabbit hole until we found a solution.  I say a solution, because I'm sure there are more elegant and efficient ways of gathering this data.  The main issue in this current implementation is that collecting the rating info takes several seconds, which isn't ideal - but it's better than not having rating information.  

Here's a sneak peak at the gem in action.

![daily_recipe](http://imgur.com/5P2nZZw.png)

The gem, [daiy_recipe](https://rubygems.org/gems/daily_recipe), is my first gem.  I know it's not perfect and I definitely stumbled my way through much of it, and it would have taken much longer to get the rating info integrated if it wasn't for the awesome Learn community.  For all of the difficulties in making it and the shortcomings of the gem, this is an accomplishment that I'm proud of and am really excited to share.  So the next time you want to make a tasty recipe, and you *really* want to read it from your command line, this is the gem for you.  

Happy coding (and cooking)!  




