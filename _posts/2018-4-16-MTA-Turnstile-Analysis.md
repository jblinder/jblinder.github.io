---
layout: post
Project Benson: MTA Turnstile Analysis
title:  "Project 1 - MTA  Turnstile Analysis"
published: false
---
![Viz 3]({{ "/images/project01/mta-style.jpg" | absolute_url }})

## Introduction
For project 1, we were tasked with helping WomenTechWomenYes (WTWY)-- a new, inclusive tech organization-- increase participation for their upcoming annual gala. WTWY is planning to deploy street teams to get email sign-ups among NYC subway commuters. The goal is to come up with a strategy that will help these street teams maximize sign ups.


### The Dataset
The data we used is from the [MTA Turnstile dataset](http://web.mta.info/developers/turnstile.html). This dataset contains hundreds of thousands of entries, each representing the total entries and exits over a 4 hour period for an individual MTA turnstile. The variables it contains are:
* **C/A** - Control Area (A002)
* **UNIT** - Remote Unit for a station (R051)
* **SCP** - Subunit Channel Position represents an specific address for a device (02-00-00)
* **STATION** - Represents the station name the device is located at
* **LINENAME** - Represents all train lines that can be boarded at this station
           Normally lines are represented by one character.  LINENAME 456NQR repersents train server for 4, 5, 6, N, Q, and R trains.
* **DIVISION** - Represents the Line originally the station belonged to BMT, IRT, or IND   
* **DATE** - Represents the date (MM-DD-YY)
* **TIME** - Represents the time (hh:mm:ss) for a scheduled audit event
* **DESC** - Represent the "REGULAR" scheduled audit event (Normally occurs every 4 hours)
           1. Audits may occur more that 4 hours due to planning, or troubleshooting activities. 
           2. Additionally, there may be a "RECOVR AUD" entry: This refers to a missed audit that was recovered. 
* **ENTRIES** - The cumulative entry register value for a device
* **EXIST** - The cumulative exit register value for a device


## Process

### Timeframe
The timeframe we focus on was from the end of April to the beginning of June 2017-- Roughly a months-worth of data. We thought this would be a large enough sample to see patterns, and could iterate to include more data if time permitted.


### Data Wrangling
We initially noticed many challenges and issues in the data. Among these were:
* Counters are cumulative, so finding accurate counts involves subtracting entry/ exit amounts from each other
* Some column names are poorly formatted
* Counters jump and (might)reset
* There are duplicate indicies 
* There are some negative values that might indicate a turnstile is turning in reverse
* Exit data is very different from entry data, which might mean a portion of people are leaving through exit doors.
* Entries and exits are logged every 4 hours, so the data isn't too granular. Also, some of the log times are inconsistent.

Our first step was to remove outliers in the data. We visualize the distribution of the entries using a scatterplot to see determine how much they varied.

![Horizontal Bar Chart]({{ "/images/project01/outlier.png" | absolute_url }})

```python
fig, ax = plt.subplots(figsize=(20,10))
cm = plt.cm.get_cmap('CMRmap')
xy = np.arange(100)
z = xy
s = ax.scatter(x='entries_delta', y=df['entries_delta'].index,data=df, alpha=.6);
```
Based on this chart, we experimented with different maximum thresholds to focus on for our analysis.

We also realized that filtering data by a `STATION` identifier wouldn't be accurate, since there are often multiple stations on different train lines with the same name (i.e. there are 5 different 23rd street stations). To remedy this we created a new column called `STATIONID` that concatanated the station name and linename:

```python
df = df.sort_values('LINENAME')
df["STATIONID"] = df["STATION"] + " " +  df["LINENAME"]
df
```

### User Story
The goal of this project is to optimize the success of engaging people in conversations. The dataset we're dealing with can give us a bunch of insight, but data is only useful when it's contextualized. 

To this end, we developed a user story for a single commuter (Samantha), and tried to imagine what her daily commute might look like. 

> Samantha is a software developer and lives in Brooklyn, NY. She works for a startup on 42nd St. and Park Ave.— her days at the office are long. Two years ago, Samantha started a feminist network security meetup in Bushwick with her roommate. She likes tacos, cats, and well-executed puns.

Samantha's morning commute starts at the Morgan Avenue L train. She transfers at 14th Street Union Square to the 6, and takes it uptown to 42 Street Grand Central, where she works.

One of our first assumptions was that transfer hubs (i.e. Union Square)  would be an ideal place to send street team members to get sign ups because they're highly saturated. However, after looking at Samantha's journey, we realized she would likely be in a rush to catch the 6 train, and wouldn't be free to have a chat with street team members. The user story, and it's micro focus, ensured that we didn't oversimplify our strategies by solely taking a bird's eye view of the problem.

## Analysis

### Top stations
Our first analysis was to find the stations with the highest saturation of traffic. 
```python
# `entries_delta` is a column that contains the difference between consecutive turnstile counts
df_top = df.groupby(['STATIONID']).sum()
df_top = df_top.sort_values(by=['entries_delta'],ascending=False).head(5)
```

![Horizontal Bar Chart]({{ "/images/project01/hbar.png" | absolute_url }})

We visualized this data using a horizontal bar chart and realized that the top 6 stations, 5 of them were within the bounds of Park Ave <--> 8th Ave & 34th St. <--> 42nd St.



### Weekend vs. Weekday
One question we had while analyzing the data was whether weekend or weekday subway traffic would be higher. We added a new column to the dataframe that specifies whether a specific row occured on a weekday or weekend:

```python
def to_wknd_wkdy(date):
    day_of_week = int(datetime.strftime(date, '%w'))
    return 'Weekend' if day_of_week in [0,6] else 'Weekday'

df['date_new'] = df['date_str'].apply(lambda x: datetime.strptime(x, '%m/%d/%Y %H:%M:%S'))
df['wknd_wkdy'] = df['date_new'].apply(to_wknd_wkdy)
```

After this, we were able to plot the difference between the average entries, over the course of a day, on the weekends and weekdays for the top 5 most frequented stations.

![Weekday/Weekend]({{ "/images/project01/weekendday.png" | absolute_url }})

We realized weekday traffic was overall higher than the weekends, so we decided to focus our attention of weekday travel.

### Consistency

After we had the top 5 stations, we were curious if they maintained their rank over the course of our month-long sample. We used a stacked line chart to measure this:

![Viz 3]({{ "/images/project01/top5.png" | absolute_url }})

We found that, for the most part, these stations maintained rank over the course of the sample. One interesting exception was on May 7th, 2017 which turned out to be the 5 burrough bike tour-- there were events at different stations around the city which may have accounted for this discrepancy.

### Case Study: Grand Central Station

To find out more about the configuration of stations we analyzed one of the most frequented control areas in the dataset, `R238` in Grand Central. We found an internal document on the MTA website that maps out this area (in red).

![Viz 3]({{ "/images/project01/gc1.png" | absolute_url }})

Given the high amount of foot traffic for this control area, the space was fairly small. It would likely be difficult to engage with people in a conversation underground, however the corridor upstairs has much more space.

![Viz 3]({{ "/images/project01/gc3.png" | absolute_url }})

This document also highlighted the large amount of stairwells in this station (in green). One strategy we were considering was having members hand out information at stairwells, as they would be able to engage with individuals in a more open space than turnstiles. Commuters hands would be likely free than at the turnstiles, where they would have to pull out their metro card to swipe through. After reviewing this document, It seemed that having street team members in station stairwells might not be ideal because there are so many stairwells to account for-- there would have to be a 
street team member at each one.

![Viz 3]({{ "/images/project01/gc2.png" | absolute_url }})

## Strategies

Based on our analyses, we developed 3 concrete strategies to optimize the WTWY street team's strategy.

**1 - Location**

Focus on a section of the cities with the most saturated stations, Midtown South-- Between  Park Ave <--> 8th Ave & 34th St. <--> 42nd St. In this area, overall exits exceeded entries in the morning (between 8am-12pm), and entries exceeded exits in the evening (between 8pm-12am). We concluded that this was a very dense commercial hub that people commute to for work, and could be an ideal place to target commuters throughout the day (given how long they spend in the region).

![Map]({{ "/images/project01/map.png" | absolute_url }})

**2 - Placement**
When we developed our user story, we realized that times and stations with high volume weren't necessarily ideal to get sign-ups, since it's very difficult to engage with people in a crowded area. Instead, we decided to split the street team into two separate teams:

* Team A focus on collecting email addresses and will be placed at top stations where volume is low.
* Team B focus on handing out information about WTWY (and free USB sticks) and will be placed in top stations where volume is high.

We believe this strategy will increase both engagement and awareness, and address each in an ideal setting.

**3 - Citi Bike**
We also considered how Citi Bike stations might allow us to engage with commuters that travel to areas that are inaccessible by subway. 

We looked at existing analyses of Citi Bike data and realized the hourly usage matched up perfectly with our existing insights. 

![Citi Bike]({{ "/images/project01/comparison.png" | absolute_url }})

We plan to place street team members at Citi Bike stations next to subways that border our area of interest. This would help intercept commuters that are coming from or leaving areas that are inaccessible by subway. It would also allow the street teams engage with commuters in an open, less crowded setting, which may increase engagement.

## Next steps

As we progressed through our approach, we kept track of false assumptions we might be making or key variables we might be overlooking. A few of these were:
* We silo our focus to South Midtown-- this narrow focus might be overlooking key outliers.

* Our primary region is mostly commercial-- could commuter's moods, and propensity to sign up, be higher in residential areas?

* We didn't focus on weekends— perhaps people might not be as busy, and thus more willing to participate.

We were hoping to implement more datasets, but ran out of time. The two main statistics were were interested in including were:
* Donation statistics-- where are people more likely to donate? A lot of research suggests that wealthier people donate less-- how would this factor into our proposed strategy?
* Reliable demographic data-- in 2016, over 5.5 million people took the subway on an average weekday. Given how diverse New York City is, we felt that using population demographic data would make too many assumptions about who took the subway. However, we were curious if there were datasets that might be more reliable and make less assumptions about populations we are focusing on.

Link to PDF presentation: [get the PDF]({{ "/images/project01/presentation-mta-tech_jblinder_dlee.a.pdf" | absolute_url }})
Team:  
* Justin Blinder
* Doug Lee


