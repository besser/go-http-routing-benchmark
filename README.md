Go HTTP Router Benchmark
========================

This benchmark suite aims to compare the performance of HTTP request routers for [Go](https://golang.org) by implementing the routing structure of some real world APIs.
Some of the APIs are slightly adapted, since they can not be implemented 1:1 in some of the routers.

Of course the tested routers can be used for any kind of HTTP request → handler function routing, not only (REST) APIs.


#### Tested routers & frameworks:

 * [Bone](https://github.com/go-zoo/bone)
 * [Gorilla Mux](http://www.gorillatoolkit.org/pkg/mux)
 * [Pat](https://github.com/bmizerany/pat)


## Motivation

Go is a great language for web applications. Since the [default *request multiplexer*](http://golang.org/pkg/net/http/#ServeMux) of Go's net/http package is very simple and limited, an accordingly high number of HTTP request routers exist.

Unfortunately, most of the (early) routers use pretty bad routing algorithms. Moreover, many of them are very wasteful with memory allocations, which can become a problem in a language with Garbage Collection like Go, since every (heap) allocation results in more work for the Garbage Collector.

Lately more and more bloated frameworks pop up, outdoing one another in the number of features. This benchmark tries to measure their overhead.

Beware that we are comparing apples to oranges here, we compare feature-rich frameworks to packages with simple routing functionality only. But since we are only interested in decent request routing, I think this is not entirely unfair. The frameworks are configured to do as little additional work as possible.

If you care about performance, this benchmark can maybe help you find the right router, which scales with your application.


## Results

Benchmark System:
 * Intel Core i5 (4x 1,60GHz)
 * 2x 2 GiB DDR3-1333 RAM, dual-channel
 * go version go1.6 darwin/amd64
 * OS X 10.11.3 (15D21) (Darwin Kernel 15.3.0)


### Memory Consumption

Besides the micro-benchmarks, there are 3 sets of benchmarks where we play around with clones of some real-world APIs, and one benchmark with static routes only, to allow a comparison with [http.ServeMux](http://golang.org/pkg/net/http/#ServeMux).
The following table shows the memory required only for loading the routing structure for the respective API.
The best values for each test are bold.

| Router       | Static    | GitHub     | Google+   | Parse     |
|:-------------|----------:|-----------:|----------:|----------:|
| HttpServeMux |  17824 B  |         -  |        -  |        -  |
| Bone         |  37872 B  |   97696 B  |   6448 B  |  10992 B  |
| Gorilla Mux  | 668496 B  | 1493280 B  |  70976 B  | 122136 B  |
| Pat          |__20464 B__| __21200 B__| __1856 B__| __2560 B__|

The first place goes to [Pat](https://github.com/bmizerany/pat), followed by [Bone](https://github.com/go-zoo/bone) and [Gorilla Mux](http://www.gorillatoolkit.org/pkg/mux). Now, before everyone starts reading the documentation of Pat, `[SPOILER]` this low memory consumption comes at the price of relatively bad routing performance. The routing structure of Pat is simple - probably too simple. `[/SPOILER]`.

Moreover main memory is cheap and usually not a scarce resource. As long as the router doesn't require Megabytes of memory, it should be no deal breaker. But it gives us a first hint how efficient or wasteful a router works.


### Static Routes

The `Static` benchmark is not really a clone of a real-world API. It is just a collection of random static paths inspired by the structure of the Go directory. It might not be a realistic URL-structure.

The only intention of this benchmark is to allow a comparison with the default router of Go's net/http package, [http.ServeMux](http://golang.org/pkg/net/http/#ServeMux), which is limited to static routes and does not support parameters in the route pattern.

In the `StaticAll` benchmark each of 157 URLs is called once per repetition (op, *operation*). If you are unfamiliar with the `go test -bench` tool, the first number is the number of repetitions the `go test` tool made, to get a test running long enough for measurements. The second column shows the time in nanoseconds that a single repetition takes. The third number is the amount of heap memory allocated in bytes, the last one the average number of allocations made per repetition.

The logs below show, that http.ServeMux has only medium performance, compared to more feature-rich routers. The fastest router only needs 1.8% of the time http.ServeMux needs.

```
BenchmarkHttpServeMux_StaticAll 	    1000	 1266940 ns/op	      96 B/op	       8 allocs/op

BenchmarkBone_StaticAll         	   10000	  139700 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_StaticAll   	     500	 3113180 ns/op	   70432 B/op	    1107 allocs/op
BenchmarkPat_StaticAll          	     500	 3246139 ns/op	  533906 B/op	   11123 allocs/op
```

### Micro Benchmarks

The following benchmarks measure the cost of some very basic operations.

In the first benchmark, only a single route, containing a parameter, is loaded into the routers. Then a request for a URL matching this pattern is made and the router has to call the respective registered handler function. End.
```
BenchmarkBone_Param             	 1000000	    2075 ns/op	     384 B/op	       3 allocs/op
BenchmarkGorillaMux_Param       	  300000	    5435 ns/op	     752 B/op	       8 allocs/op
BenchmarkPat_Param              	  300000	    4187 ns/op	     648 B/op	      12 allocs/op
```

Same as before, but now with multiple parameters, all in the same single route. The intention is to see how the routers scale with the number of parameters. The values of the parameters must be passed to the handler function somehow, which requires allocations. Let's see how clever the routers solve this task with a route containing 5 and 20 parameters:
```
BenchmarkBone_Param5            	  500000	    3321 ns/op	     432 B/op	       3 allocs/op
BenchmarkGorillaMux_Param5      	  200000	    9071 ns/op	     816 B/op	       8 allocs/op
BenchmarkPat_Param5             	  200000	    9544 ns/op	     964 B/op	      32 allocs/op

BenchmarkBone_Param20           	  100000	   13717 ns/op	    2540 B/op	       5 allocs/op
BenchmarkGorillaMux_Param20     	  100000	   19827 ns/op	    2923 B/op	      10 allocs/op
BenchmarkPat_Param20            	   30000	   47499 ns/op	    4687 B/op	     111 allocs/op
```

Now let's see how expensive it is to access a parameter. The handler function reads the value (by the name of the parameter, e.g. with a map lookup; depends on the router) and writes it to our [web scale storage](https://www.youtube.com/watch?v=b2F-DItXtZs) (`/dev/null`).
```
BenchmarkBone_ParamWrite        	 1000000	    2280 ns/op	     384 B/op	       3 allocs/op
BenchmarkGorillaMux_ParamWrite  	  200000	    5915 ns/op	     752 B/op	       8 allocs/op
BenchmarkPat_ParamWrite         	  200000	    6789 ns/op	    1072 B/op	      17 allocs/op
```

### [Parse.com](https://parse.com/docs/rest#summary)

Enough of the micro benchmark stuff. Let's play a bit with real APIs. In the first set of benchmarks, we use a clone of the structure of [Parse](https://parse.com)'s decent medium-sized REST API, consisting of 26 routes.

The tasks are 1.) routing a static URL (no parameters), 2.) routing a URL containing 1 parameter, 3.) same with 2 parameters, 4.) route all of the routes once (like the StaticAll benchmark, but the routes now contain parameters).

Worth noting is, that the requested route might be a good case for some routing algorithms, while it is a bad case for another algorithm. The values might vary slightly depending on the selected route.

```
BenchmarkBone_ParseStatic       	 1000000	    1299 ns/op	     144 B/op	       3 allocs/op
BenchmarkGorillaMux_ParseStatic 	  300000	    5370 ns/op	     448 B/op	       7 allocs/op
BenchmarkPat_ParseStatic        	 1000000	    1580 ns/op	     240 B/op	       5 allocs/op

BenchmarkBone_ParseParam        	  500000	    2600 ns/op	     464 B/op	       4 allocs/op
BenchmarkGorillaMux_ParseParam  	  200000	    6285 ns/op	     752 B/op	       8 allocs/op
BenchmarkPat_ParseParam         	  200000	    5873 ns/op	    1120 B/op	      17 allocs/op

BenchmarkBone_Parse2Params      	 1000000	    2434 ns/op	     416 B/op	       3 allocs/op
BenchmarkGorillaMux_Parse2Params	  200000	    7801 ns/op	     768 B/op	       8 allocs/op
BenchmarkPat_Parse2Params       	  200000	    5978 ns/op	     832 B/op	      17 allocs/op

BenchmarkBone_ParseAll          	   30000	   57422 ns/op	    8048 B/op	      90 allocs/op
BenchmarkGorillaMux_ParseAll    	    5000	  240761 ns/op	   16560 B/op	     198 allocs/op
BenchmarkPat_ParseAll           	   10000	  124844 ns/op	   17264 B/op	     343 allocs/op
```


### [GitHub](http://developer.github.com/v3/)

The GitHub API is rather large, consisting of 203 routes. The tasks are basically the same as in the benchmarks before.

```
BenchmarkBone_GithubStatic      	   50000	   25401 ns/op	    2880 B/op	      60 allocs/op
BenchmarkGorillaMux_GithubStatic	   50000	   30047 ns/op	     448 B/op	       7 allocs/op
BenchmarkPat_GithubStatic       	  100000	   22100 ns/op	    3648 B/op	      76 allocs/op

BenchmarkBone_GithubParam       	  200000	   11951 ns/op	    1456 B/op	      16 allocs/op
BenchmarkGorillaMux_GithubParam 	  100000	   18025 ns/op	     768 B/op	       8 allocs/op
BenchmarkPat_GithubParam        	  100000	   15353 ns/op	    2464 B/op	      48 allocs/op

BenchmarkBone_GithubAll         	     300	 4268500 ns/op	  548738 B/op	    7241 allocs/op
BenchmarkGorillaMux_GithubAll   	     100	10511628 ns/op	  144464 B/op	    1588 allocs/op
BenchmarkPat_GithubAll          	     200	 8484392 ns/op	 1499581 B/op	   27435 allocs/op
```

### [Google+](https://developers.google.com/+/api/latest/)

Last but not least the Google+ API, consisting of 13 routes. In reality this is just a subset of a much larger API.

```
BenchmarkBone_GPlusStatic       	 5000000	     383 ns/op	      32 B/op	       1 allocs/op
BenchmarkGorillaMux_GPlusStatic 	  500000	    4024 ns/op	     448 B/op	       7 allocs/op
BenchmarkPat_GPlusStatic        	 2000000	     696 ns/op	      96 B/op	       2 allocs/op

BenchmarkBone_GPlusParam        	 1000000	    2224 ns/op	     384 B/op	       3 allocs/op
BenchmarkGorillaMux_GPlusParam  	  200000	    7228 ns/op	     752 B/op	       8 allocs/op
BenchmarkPat_GPlusParam         	  300000	    4192 ns/op	     688 B/op	      12 allocs/op

BenchmarkBone_GPlus2Params      	  200000	    5782 ns/op	     736 B/op	       7 allocs/op
BenchmarkGorillaMux_GPlus2Params	  100000	   15006 ns/op	     768 B/op	       8 allocs/op
BenchmarkPat_GPlus2Params       	  100000	   12453 ns/op	    2256 B/op	      34 allocs/op

BenchmarkBone_GPlusAll          	   50000	   37427 ns/op	    4912 B/op	      61 allocs/op
BenchmarkGorillaMux_GPlusAll    	   10000	  113136 ns/op	    9248 B/op	     102 allocs/op
BenchmarkPat_GPlusAll           	   10000	  122563 ns/op	   16576 B/op	     298 allocs/op
```


## Conclusions
First of all, there is no reason to use net/http's default [ServeMux](http://golang.org/pkg/net/http/#ServeMux), which is very limited and does not have especially good performance. There are enough alternatives coming in every flavor, choose the one you like best.

Secondly, the broad range of functions of some of the frameworks comes at a high price in terms of performance. For example Martini has great flexibility, but very bad performance. Martini has the worst performance of all tested routers in a lot of the benchmarks. Beego seems to have some scalability problems and easily defeats Martini with even worse performance, when the number of parameters or routes is high. I really hope, that the routing of these packages can be optimized. I think the Go-ecosystem needs great feature-rich frameworks like these.

Last but not least, we have to determine the performance champion.

Denco and its predecessor Kocha-urlrouter seem to have great performance, but are not convenient to use as a router for the net/http package. A lot of extra work is necessary to use it as a http.Handler. [The README of Denco claims](https://github.com/naoina/denco/blob/b03dbb499269a597afd0db715d408ebba1329d04/README.md), that the package is not intended as a replacement for [http.ServeMux](http://golang.org/pkg/net/http/#ServeMux).

[Goji](https://github.com/zenazn/goji/) looks very decent. It has great performance while also having a great range of features, more than any other router / framework in the top group.

Currently no router can beat the performance of the [HttpRouter](https://github.com/julienschmidt/httprouter) package, which currently dominates nearly all benchmarks.

In the end, performance can not be the (only) criterion for choosing a router. Play around a bit with some of the routers, and choose the one you like best.

## Usage

If you'd like to run these benchmarks locally, you'll need to install the packge first:

```bash
go get github.com/julienschmidt/go-http-routing-benchmark
```
This may take a while due to the large number of dependencies that need to be downloaded. Once that command completes, you can run the full set of benchmarks like this:

```bash
cd $GOPATH/src/github.com/julienschmidt/go-http-routing-benchmark
go test -bench=.
```

> **Note:** If you run the tests and it SIGQUIT's make the go test timeout longer (#44)
>
>```
go test -timeout=2h -bench=.
```


You can bench specific frameworks only by using a regular expression as the value of the `bench` parameter:
```bash
go test -bench="Martini|Gin|HttpMux"
```
