Context
-------

Each summer I volunteer at a local country music festival organized by a family friend. The event has grown substantially since its creation in 2010. Over time, the event organizers have begun to incorporate more entertainment and amenities to guests (VIP passes, more artists, etc.). Prices have increased to accomodate the costs of a bigger, higher-demand festival.

One of the concert's major additions since inception is an electronic ticketing system similar to what you'd see at a large sporting event. Attendees now scan their tickets at a kiosk outside the event to receive a wristband & gain entry to the festival grounds.

As this music festival becomes a more 'premium' event, it's important to keep concertgoers happy by offering a high level of service. The first interaction they have is this ticket kiosk. I think a keeping wait times under 5 minutes would be excellent.

To design a system capable of handling the 'rush' (without over-staffing the kiosk), I thought it would be fun to experiment with discrete event simulation in R to simulate the check-in system over time.

``` r
library(simmer)
library(simmer.plot)
library(parallel)
library(dplyr)
library(ggplot2)
```

Data
----

I don't have data from the electronic ticketing system, so instead I made up some fake data based on my experience at the concert.

I had to create two 'distributions' to simulate the process:

1.  The time required to scan a ticket, apply a wristband, and address any complications/questions
2.  The time between guest arrivals in the queue

### Ticket Scanning Time

I assumed ticket scanning times looked something like the distribution I created below.

It usually takes around one minute per person to scan their ticket & apply the correct wristband. Occasionally it takes less time, but no less than 45 seconds. Every now and again there will be an issue with a ticket or a complication that increases the time to as much as 5 minutes, but this is rare.

``` r
Sample_Scan_Times = rexp(10000,1.5) %>% 
  ifelse(. < .75,.75,.)

ggplot() + 
  geom_histogram(aes(x = Sample_Scan_Times)) + 
  geom_vline(aes(xintercept = mean(Sample_Scan_Times))) +
  labs(x = "Elapsed Time (Minutes)",
       y = "Frequency",
       title = "Time Required to Scan Ticket, Apply Wristband, and Answer Questions")
```

<img src="https://github.com/MatthewBrower/QueuingWorkflow/tree/master/Event_Simulation_files/figure-markdown_github/unnamed-chunk-1-1.png">

### Guest Intra-Arrival Time

For the rate of guest arrivals, I relied on my experience working at the kiosk as a volunteer - but these assumptions are probably not reflective of the slower times toward the beginning and end of each day.

I'm sure I could incorporate fluctuations in arrival rate into this workflow, but for the sake of simplicity I limited the scope of my simulations to the peak 4 hours of concert time (which will be integrated in the later steps of the simulation below).

``` r
Guest_Arrival_Intervals = rexp(10000,2) %>% ifelse(. < .2,.2,.) 

ggplot() + 
  geom_histogram(aes(x = Guest_Arrival_Intervals)) + 
  geom_vline(aes(xintercept = mean(Guest_Arrival_Intervals))) +
  labs(x = "Elapsed Time (Minutes)",
       y = "Frequency",
       title = "Time Between Attendee Arrivals")
```

![](Event_Simulation_files/figure-markdown_github/unnamed-chunk-2-1.png) \#\# Simulations with `simmer`

I discovered the `simmer` package in R through some Googling and decided to give it a try. My only previous experience with queue modeling was done through SAS Simulation Studio; I found this package to be a nice open source substitute.

Without going into too much detail here, `simmer` can visualize queue processes and then run simulations based on the structures (servers, lines, etc.) you create.

My ticket kiosk is simple: it's just a one step process with a single queue. What I've done here is establish a `simmer` environment and set up the structure (one 'seize' activity for ticket scanning + wristbands, which 'releases' after a certain amount of time).

``` r
env <- simmer("Country_Festival")

Queue <- trajectory("Concert Attendees") %>%
  ## add an intake activity 
  seize("Booth", 1) %>%
  ## add the timing data generator here:
  timeout(function() rexp(1,1.5) %>% 
  ifelse(. < .75,.75,.)) %>%
  ## release once ticketing is done
  release("Booth", 1)
```

Here's that that looks like when I plot it.

``` r
plot(Queue)
```

<!--html_preserve-->

<script type="application/json" data-for="htmlwidget-5d619bed012dc8a9bcba">{"x":{"diagram":"digraph {\n\nnode [fontname = \"sans-serif\",\n     width = \"1.5\"]\n\n\n  \"1\" [label = \"Seize\", shape = \"box\", style = \"filled\", color = \"#7FC97F\", tooltip = \"resource: Booth, amount: 1\"] \n  \"2\" [label = \"Timeout\", shape = \"box\", style = \"solid\", color = \"black\", tooltip = \"delay: 0x7ff66c503950\"] \n  \"3\" [label = \"Release\", shape = \"box\", style = \"filled\", color = \"#7FC97F\", tooltip = \"resource: Booth, amount: 1\"] \n\"1\"->\"2\" [id = \"1\", color = \"black\", style = \"solid\"] \n\"2\"->\"3\" [id = \"2\", color = \"black\", style = \"solid\"] \n}","config":{"engine":null,"options":null}},"evals":[],"jsHooks":[]}</script>
<!--/html_preserve-->
Once I set up the environment itself, it was pretty easy to add an arrival generator for guests and to add 'resources' specify the number of servers at the kiosk. Simulations can be run using the `run` command, which I've done below. The output from this shows the status of the Ticket Kiosk at the end of a 4 hour run.

``` r
env %>% 
  reset(.) %>% 
  add_resource("Booth", 2) %>%
  add_generator("Guest", Queue, function() rexp(1,2) %>% ifelse(. < .2,.2,.)) %>%
  run(until = 240)
```

    ## simmer environment: Country_Festival | now: 240 | next: 240.087211414182
    ## { Resource: Booth | monitored: TRUE | server status: 2(2) | queue status: 1(Inf) }
    ## { Generator: Guest | monitored: 1 | n_generated: 437 }

Running this simulation to check on its status is great, but running multiple trials of this and recording the results would be even better. Great news: Simmer also provides the option of running many of the same simulations in parallel, which is what I've done below. Here's a look at 50 simulations with 3 servers at the kiosk.

``` r
envs <- mclapply(1:50, function(i) {
  simmer("Country_Festival") %>%
    add_resource("Booth", 3) %>%
    add_generator("Guest", Queue, function() rexp(1,2) %>% ifelse(. < .2,.2,.) ) %>%
    run(240) %>%
    wrap()
})
```

Storing those runs to the `envs` object preserves the data from each run, and the package has some pretty slick data visualization capabilities built in.

When I run this with 3 servers, it seems like the system is (for the most part) stable - wait times are usually below 5 minutes, and when they do exceed 5 minutes they seem to successfully recover over time.

``` r
plot(envs, what = "resources", metric = "utilization", c("Booth"))
```

![](Event_Simulation_files/figure-markdown_github/unnamed-chunk-7-1.png)

``` r
plot(envs, what = "resources", metric = "usage", c("Booth"), items = "server")
```

![](Event_Simulation_files/figure-markdown_github/unnamed-chunk-7-2.png)

``` r
plot(envs, what = "arrivals", metric = "flow_time")
```

    ## `geom_smooth()` using method = 'gam'

![](Event_Simulation_files/figure-markdown_github/unnamed-chunk-7-3.png)

I tried to see if I could get away with only staffing 2 servers, but the wait times were too long for the booth to be successful. On average, flow time is still below 5 minutes throughout...but there are some simulations where time climbs well above 10 minutes.

``` r
envs <- mclapply(1:50, function(i) {
  simmer("Country_Festival") %>%
    add_resource("Booth", 2) %>%
    add_generator("Guest", Queue, function() rexp(1,2) %>% ifelse(. < .2,.2,.) ) %>%
    run(240) %>%
    wrap()
})
plot(envs, what = "resources", metric = "utilization", c("Booth"))
```

![](Event_Simulation_files/figure-markdown_github/unnamed-chunk-9-1.png)

``` r
plot(envs, what = "resources", metric = "usage", c("Booth"), items = "server")
```

![](Event_Simulation_files/figure-markdown_github/unnamed-chunk-9-2.png)

``` r
plot(envs, what = "arrivals", metric = "flow_time")
```

    ## `geom_smooth()` using method = 'gam'

![](Event_Simulation_files/figure-markdown_github/unnamed-chunk-9-3.png)

So what?
--------

I think I have enough here to prove out the concept, but without actual data we don't have enough to conclude that 3 servers would be enough throughout the festival.

If we were to continue this work, I'd like to:

-   Break up the ticketing & wristband process into a few more steps (right now we've lumped it all into one). Separating out the steps would allow me to experiment with different service models to think through the efficiency of alternative arrangements (i.e. one scanner + two wristbanders, etc.)
-   Get arrival data to understand when the 'rush' truly happens, and use that to update our arrival + service time assumptions
-   Experiment with different hourly schedules and use `simmer.optimize` to help me find the best possible schedule for the event.
