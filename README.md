# Win probability of two-legged ties in soccer

Win probability models are somewhat ubiquitous in sports analysis these days, with data used to describe the back-and-forth rhythm of a particular game. Less common are models that forecast the result of an aggregate outcome over multiple games. This is one attempt at that.

The model here aims to predict the winner in a specific kind of soccer competition: The [two-legged tie](https://en.wikipedia.org/wiki/Two-legged_tie). In such matchups, each team hosts one game (referred to as a "leg") and the winner is the team that scores the most goals over both games. This specifically focuses on competitions where the ["away goals rule"](https://en.wikipedia.org/wiki/Away_goals_rule) is used to break any deadlock.

## Abstract of the model

The model is created using localized logistic regression, implemented using the `locfit` package in R. It takes a few inputs:

* The time remaining in the tie
* The goal difference between the two teams
* The away goal difference between the two teams
* The difference in the number red-carded players for a particular match
* The pre-match betting odds for the first leg of the tie

Those factors are used to spit out a probability that the team hosting the first leg will win the tie.

## Getting the data

All data on the match events comes from [fbref.com](https://fbref.com/). Betting market information comes from [oddsportal.com](http://oddsportal.com/).

Currently, this covers two competitions that both use two-legged ties and the away goals rule for certain stages of their competition: the UEFA Champions League and the UEFA Europa League (all matches except the group stages). It could likely be extended to other competitions, such as the CONMEBOL Copa Libertadores and more.

### Scraping match events

All scraping of fbref.com is done in R and uses a ["get or retrieve"](data-get/fbref/scrape-seasons.R#L13-L30) function that saves a raw copy of the entire targeted webpage on the first request, then accesses that copy on subsequent requests. This is intended to keep the number of requests to the website down and preserve the data for future access. The downside of this approach is that should the data on an already-accessed page change, it will need to be refetched. TODO: Use `override=TRUE` flag in function.

#### scrape-seasons.R

Crawls over the index pages for the CL and EL gathering info on the knockout round and qualifying round ties that need to be scraped for events, and the urls at which that information is stored.

*match-urls.csv*

| stagecode |    competition   | code_string | szn       | stage    | round      | dates                         | team1     | team2     | teamid1  | teamid2  | winner    | aggscore | result                                                       | hometeam1 | date1  | score1 | url1                                                                                             | hometeam2 | date2 | score2 | url2                                                                                          |
|:---------:|:----------------:|-------------|-----------|----------|------------|-------------------------------|-----------|-----------|----------|----------|-----------|----------|--------------------------------------------------------------|-----------|--------|--------|--------------------------------------------------------------------------------------------------|-----------|-------|--------|-----------------------------------------------------------------------------------------------|
|           | Champions League | cl          | 2018-2019 | knockout | Final      | June 1, 2019                  | Liverpool | Tottenham | 822bd0ba | 361ca564 | Liverpool | 2–0      | Liverpool won match in normal time.                          |           |        |        |                                                                                                  |           |       |        |                                                                                               |
| cl-1k-3sf | Champions League | cl          | 2018-2019 | knockout | Semifinals | April 30, 2019 to May 8, 2019 | Tottenham | Ajax      | 361ca564 | 19c3f8c4 | Tottenham | 3–3      | Tottenham won on away goals, after aggregate score was tied. | Tottenham | Apr 30 | 0–1    | https://fbref.com/en/matches/41848af6/Tottenham-Hotspur-Ajax-April-30-2019-UEFA-Champions-League | Ajax      | May 8 | 2–3    | https://fbref.com/en/matches/09773f5a/Ajax-Tottenham-Hotspur-May-8-2019-UEFA-Champions-League |

#### scrape-games.R

Starts by filtering the two-legged ties set down to just ones that have detailed match data. (For both the CL and EL this is from the 2014-15 season until now.) Crawls over the urls for each game and extracts the match events.



* scrape-teams.R: Creates an index of all teams listed by fbref.com and their countries, to join back to the data later. Can be run independently of the other scripts.

### Scraping odds

The scraping of oddsportal.com is done in R using RSelenium. That requires first firing up a docker instance with a headless Firefox browser:

```bash
sudo docker run -d -p 4445:4444 selenium/standalone-firefox:3.141.59
```

* scrape-odds.R: A lot of logic to scrape oddsportal pages, where the odds are not in the page body (they come via an XHR). Sometimes they don't show up at all! Messy code but it should capture most quirks. Dumps out just the raw HTML
* parse-odds.R: Goes through the HTML of each page to create one massive odds file. Listed American odds (-250, +500, etc.) are converted into implied probability, then a vig-free probability.

## Assembling the data



