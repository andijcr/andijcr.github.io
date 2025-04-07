+++
title = 'My Mixed Integer Programming approach for the first round of Google HashCode'
date = 2017-03-15T00:00:00+02:00
+++


## Or: how Google has a nice library for optimization problems, but the documentation kinda sucks 
### Resources

The problem statement, the datasets, this jupyter notebook can be found [here](https://github.com/andijcr/andijcr.github.io/tree/769a6a127d67f8abcd25539c5c63f9b13cf4ea91/assets/hashcode)

### The challenge

Here is a summary of the challenge proposed for the first round of hashcode 2017, a more in depth explanation can be found in the [problem pdf](data/hashcode2017_streaming_videos.pdf)

It is given a net, with a datacenter, a series of cache servers, and a series of clients (or endpoints) connected to the datacenter and to 0 or more cache servers. each connection between a client and a cache/datacenter has a constant latency, and one cache server can be connected to one or more clients.

In the datacenter there are some videos, each having a fixed dimension, available to all clients.

Having a list of request for a video from a client, what is the best way to cache these video in the cache servers?
to score a solution one sums the time saved for each request, and divides by the total number of requests

### My approach

I took the road of the optimization of an integral problem: I framed the problem in a mathematical view, with a linear function to maximize (the scoring function), a set of linear constrain to describe the problem, and a set of variables that can only assume integer values. Given this formulation, there are well known algorithms that operate on the variable to find the best solution possible for the original problem.

Google has a collection of those algorithms under this library: [Google Optimization Tools](https://developers.google.com/optimization/)
Documentation is a bit lacking, and usage is a bit difficult without a basic knowledge of the subject. 
On the plus side it comes with a python binding, and this greatly helps in a jupyter notebook

### Notes 

I did implement a complete solution, but I cannot compute the solution beyond the smallest dataset because this approach is very taxing in memory requirement and time complexity, I believe the general algorithm is O(n^4).

In this problem ```N``` is mostly the number of requests, as this usually dwarfs the number of videos and the number of caches. Each cache generates only a constrain with a cardinality of the number of videos, while each request generates a constrain of costant cardinality for each cache connected to the request's endpoint.

I think a mixed heuristic approach could provide with a great score while mantaining a very low computational cost. One possibility could be to sort the request in order of descending profitability, and to generate an optimal solution using only at most ```k``` of the most profitable requests. Found a solution, a new problem can be formulated by prepopulating the caches with this solution and a new subsets of the unused requests, to compute a new improved (but not optimal) solution, that can be used in a new iteration util all the caches are full, or a iteration threshold is achieved.

#### Techical Note

[Google Optimization Tools](https://developers.google.com/optimization/) is a c++ library that has to be compiled. Under linux it's a simple ```make install```, so this should not be a problem.

Note: Here i use the MPI solver. The CP solver can be used in the same way the mip solver is used in through this notebook. the only difference is in the instantiation, done with these calls:

    # Instantiate a CP solver.
    parameters = pywrapcp.Solver.DefaultSolverParameters()
    solver = pywrapcp.Solver("simple_CP", parameters)
    and the fact that the decision vars have to be collected in a list to be passed to a decision tree
    #the examples used CHOOSE_FIRST_UNBOUND and ASSIGN_MIN_VALUE. obsly no explanation on the meanings "-_-
    decision_builder = solver.Phase(decision_vars, solver.CHOOSE_LOWEST_MIN,
                                solver.ASSIGN_MAX_VALUE)

and the solutions are provided through a collector

    # Create a solution collector.
    collector = solver.LastSolutionCollector()
    # Add the decision variables.
    for dv in decision_vars:
        collector.Add(dv)
    # Add the objective.
    collector.AddObjective(obj_expr)
        solver.Solve(decision_builder, [objective, collector])

[this](https://developers.google.com/optimization/assignment/compare_mip_cp) page does a good job highlighting the difference of the two approach
for this problem, the mip approach is faster but requires more memory


### And now for some code

Let's start with a bit of modelling


```python
#size is the capacity of the cache in Mb, ID is an integer
class Cache:
    def __init__(self, size, ID):
        self.size = size
        self.ID = ID

#like Cache, size is dimension in Mb, ID is an integer
class Video:
    def __init__(self, size, ID):
        self.size = size
        self.ID = ID

#Endpoint represent a client that generate a request, has an ID, a latency towards the datacenter
#and a list of pairs (Cache, latency) that represents the chaches to which is connected, 
#and the latency of this connection
class Endpoint:
    def __init__(self, datacenter_latency, caches_latency, ID):
        self.ID = ID
        self.datacenter_latency = datacenter_latency
        self.caches_latency = caches_latency

#A request has a starting endpoint, a target video, 
#and a cardinality (how many times the video is requested) represented as an int
class Request:
    def __init__(self, quantity, video, endpoint):
        self.quantity = quantity
        self.video = video
        self.endpoint = endpoint
```

### TDD

Before writing the solution function, it's good to write the verifier for a solution. This will help also to define what we want as a result.

(note: of course i wrote the verifier *after* i wrote rewrote the solution function)
(note: this is far from the best way to write a verifier, i know)


```python
from math import floor
from functools import reduce

#value is the calculated score, res is a map cache:list(video)
#where each cache is paired with the videos it will contain
#requestList is the list of request as given in input to the problem, to compute the score
def verifySolution(value, res, requestList):
    #verify that the videos assigned to a cache can fit inside it
    print("all the video from the solution fit into the caches: ",
          all(
              map(lambda x: x[0].size >= sum(map(attrgetter('size'), x[1])),
                  res.items())))
    
    #compute the score by multiplying each request cardinality with the saved time 
    #(latency to datacenter minus the lowest latency to a cache containing the video)
    #and divinding by the sum of the cardinality of all requests 
    #(the result is multiplied by 1000 and rounded down as per instructions) 
    score_l = []
    for r in requestList:
        dc_l = r.endpoint.datacenter_latency
        c_l = r.endpoint.caches_latency
        saved_t = r.endpoint.datacenter_latency - min([dc_l] + list(
            map(itemgetter(1), filter(lambda x: r.video in res[x[0]], c_l))))
        score_l.append((r.quantity * saved_t, r.quantity))

    tmp = reduce(lambda a, b: (a[0] + b[0], a[1] + b[1]), score_l)
    print("actual score: ", floor(1000 * tmp[0] / tmp[1]))
    print("score from the mpiSolver: ", floor(value))
    
#a nice summary printer
def summaryPrinter(value, res):
    print('score: ', value)
    for c, vl in res.items():
        print('in cache %d are present %s' %
              (c.ID,
               ', '.join(map(lambda x: 'video%d' % (x.ID), vl))))
```

# The interesting Integer Linear Programming bit

Finally the central part. this is the sum of an afternoon of exploring and learning to use the library on this notebook, and a couple of days to debug it/make it usable.

One important bit that i noticed is this: the documentation doesn't make clear that there are multiple ways to define the problem, and that a problem written for the mpi solver can be almost painlessly (more on this later) converted to a cp solver. It is a rabbit hole, and for sure I haven't yet discovered the most idiomatic way to input the problem.


```python
from ortools.linear_solver import pywraplp
from operator import itemgetter, attrgetter

def mpiSolver(videoList, cacheList, endpointList, requestList):
    # Instantiate a MPI solver. give it a name, and choose the uderling implementation (this is the default, more can be installed)
    solver = pywraplp.Solver('Cache allocation problem',
                             pywraplp.Solver.CBC_MIXED_INTEGER_PROGRAMMING)

    #create a matrix of intvar y[cache_i, video_j], and put a constrain on space usage of a cache
    #each y can be 1 if video_j is present on cache_i, or 0 otherwise
    #cache_video will be a map cache:list(pair(video, IntVar))
    cache_video = {}
    for c in cacheList:
        cache_video_vars = {
            #IntVar is what the solver will vary to find a solution. we create it, and use it to define constraints
            #it get a range and a name for debug purposes. Note: this name should be unique, otherwise crashes happens
            v: solver.IntVar(0, 1, 'v:%d_c:%d' % (v.ID, c.ID))
            for v in videoList
        }
        #Here i'm telling the solver: the size of all the videos you decide to put in the cache should fit into the cache size.
        #notice the nice way to express it via operator overloading builtin in the library
        #solver.Sum does not suffer from recursion depth limit like the builtin sum (did not investigate the root causes tough)
        solver.Add(
            solver.Sum(
                map(lambda x: x[0].size * x[1], cache_video_vars.items())) <=
            c.size)
        cache_video[c] = cache_video_vars
        
        
    #request_min_latency is a support intvar that represents for each request 
    #the minimum latency between all the caches (and datacenter) connected to the endpoint that contains the video
    #req_var is a map request:IntVar
    req_var = {}
    for r in requestList:
        dc_L = r.endpoint.datacenter_latency
        #here again i use a IntVar. i could use a NumVar to represent a var that can have Real (instead of Int) values
        #but this way it's faster, and for this problem the found solution it's fine
        #the range is from 0 to the datacenter latency of the request's endpoint
        req_min_latency = solver.IntVar(0, dc_L, 'v:%d_e:%d' %
                                        (r.video.ID, r.endpoint.ID))

        #the tricky constrain to express as a mathematic disequation is the selection of the lowest latency
        #i used slack variables collected in this list, and the Big M tecnique
        #with this constrain i want to express that the request's latency is bigger than the caches latency.
        #given that in the end i want to maximize the negation of this variable, this set of constrains will assure that
        #req_min_latency will assume the value of the smallest latency among the caches that contains its video
        min_cache_select = []
        for c, c_l in r.endpoint.caches_latency:
            #IntVar for video present in this cache
            y = cache_video[c][r.video]
            #Support IntVar to select the lowest latency among caches
            b = solver.IntVar(0, 1, 'e:%d_v:%d_c:%d' %
                              (r.endpoint.ID, r.video.ID, c.ID))
            #Big_M (variation): if video in cache, req_min_latency is tied to this latency, 
            #otherwise it is tied to the datacenter latency
            # b if 1 trasform this costrain in a tautology (so it doesn't matter anymore)
            solver.Add(req_min_latency >= c_l + (1 - y) *
                       (dc_L - c_l) - dc_L * b)
            min_cache_select.append(b)

        if len(r.endpoint.caches_latency) == 0:
            #base case: no cache present or used
            #this one is redundant in most cases
            b = solver.IntVar(0, 1,
                              'e:%d_v:%d_dc' % (r.endpoint.ID, r.video.ID))
            solver.Add(req_min_latency >= dc_L - dc_L * b)
            min_cache_select.append(b)

        #Big_M for select: this will ensure that the minimun latency will be chosen, only one 'b' can be 0 in this set
        solver.Add(solver.Sum(min_cache_select) == (len(min_cache_select) - 1))
        #save the latency var to use it in the objective function
        req_var[r] = req_min_latency

    #the objective is to maximize the scoring function, and this is tied to the request latency (tied to the presence of a video in a cache) 
    solver.Maximize(
        solver.Sum(
            map(lambda r: r[0].quantity * (r[0].endpoint.datacenter_latency - r[1]),
                req_var.items())) *
        (1000 / sum(map(attrgetter('quantity'), req_var))))

    #this is where the magic happens. warning, high memory and hight cpu time usage ahead
    solver.Solve()

    #finally, return the computed score, and a map cache:list(videos) to represent video's allocation inside the caches
    return solver.Objective().Value(), {
        c: list(
            map(
                itemgetter(0),
                filter(lambda x: x[1].solution_value() == 1, vl.items())))
        for c, vl in cache_video.items()
    }
```

### A test dataset

A small example to test the code I've written. This is taken from the [problem pdf](data/hashcode2017_streaming_videos.pdf)

In the datacenter there are 5 videos, there are 3 caches of 100 mb each, and 2 endpoints connect to the datacenter. Only the first endpoint is connected to all the caches.

Next we have the requests that will decide the score of my solution


```python
videoList = [
    Video(ID=0, size=50), Video(ID=1, size=50), Video(ID=2, size=80),
    Video(ID=3, size=30), Video(ID=4, size=110)
]
cacheList = [
    Cache(ID=0, size=100), Cache(ID=1, size=100), Cache(ID=2, size=100)
]
endpointList = [
    Endpoint(
        ID=0,
        datacenter_latency=1000,
        caches_latency=[(cacheList[0], 100), (cacheList[1], 200), 
                        (cacheList[2], 300)]),
    Endpoint(ID=1, datacenter_latency=500, caches_latency=[])
]
requestList = [
    Request(quantity=1000, video=videoList[3], endpoint=endpointList[1]),
    Request(quantity=1500, video=videoList[3], endpoint=endpointList[0]),
    Request(quantity=500, video=videoList[4], endpoint=endpointList[0]),
    Request(quantity=1000, video=videoList[1], endpoint=endpointList[0])
]
```

### The moment of truth

if all goes well nothing will explode (a lot of things exploded before arriving at this point)


```python
value, res = mpiSolver(videoList, cacheList, endpointList, requestList)
verifySolution(value, res, requestList)
summaryPrinter(value, res)
```

    all the video from the solution fit into the caches:  True
    actual score:  562500
    score from the mpiSolver:  562500
    score:  562500.0
    in cache 1 are present video1, video3
    in cache 0 are present video1, video3
    in cache 2 are present video1, video3


Nice! the execution time for the above cell on my laptop is ~70 ms.
In contrast, in the first iteration i implemented the CP solver and i got ~3000 ms of execution time

Notice that for the scoring function only cache 0 is important, has it has the lowest latency to the endpoint.
also, if you look at the test dataset you'll see that video4 is too big to fit in a cache, so it was not considerate for the solution. 

now it's time to test a dataset. Here i implement a function to load a dataset from file (as in the problem specification)


```python
from operator import attrgetter
from itertools import groupby
from functools import reduce

#not the best way to do it
def readFromFile(filename):
    datain = open(filename)
    header = datain.readline()
    #nice trick to extract formatted input into variables
    n_vid, n_endp, n_req, n_caches, size_caches = [
        f(i) for f, i in zip([int, int, int, int, int], header.split())
    ]
    cacheList = [Cache(ID=id, size=size_caches) for id in range(n_caches)]
    vids = datain.readline()
    videoList = [
        Video(ID=id, size=int(s)) for id, s in zip(range(n_vid), vids.split())
    ]
    endpointList = []
    for id in range(n_endp):
        descr = datain.readline()
        dc_lat, c_cach = [f(i) for f, i in zip([int, int], descr.split())]
        caches = []
        for c in range(c_cach):
            cache_descr = datain.readline()
            c_id, c_lat = [
                f(i) for f, i in zip([int, int], cache_descr.split())
            ]
            caches.append((cacheList[c_id], c_lat))
        endpointList.append(
            Endpoint(ID=id, caches_latency=caches, datacenter_latency=dc_lat))
    requestList = []
    for _ in range(n_req):
        req_descr = datain.readline()
        vid, endp, quant = [
            f(i) for f, i in zip((int, int, int), req_descr.split())
        ]
        requestList.append(
            Request(
                quantity=quant,
                endpoint=endpointList[endp],
                video=videoList[vid]))

    #the tricky part: turns out that in the input dataset the requests are not completely aggregated
    #by this i mean that there are multiple requests with the same (endpoint,video) pair.
    #here i aggregate them, since mathematically the result is the same, and the solver likes it better this way
    req_comp = attrgetter('video.ID', 'endpoint.ID')
    rsorted=[reduce(lambda r1, r2: Request(video=r1.video, endpoint=r1.endpoint, quantity=r1.quantity+r2.quantity), g) for _, g in groupby(sorted(requestList, key=req_comp), key=req_comp)]

    return dict(
        caches=cacheList,
        videos=videoList,
        endpoints=endpointList,
        requests=rsorted)
```


```python
#load the smallest dataset, 78kb
small_problem = readFromFile('me_at_the_zoo.in')
```


```python
value, res = mpiSolver(small_problem['videos'], small_problem['caches'],
                       small_problem['endpoints'], small_problem['requests'])
verifySolution(value, res, small_problem['requests'])
summaryPrinter(value, res)
```

    all the video from the solution fit into the caches:  True
    actual score:  516557
    score from the mpiSolver:  516557
    score:  516557.9336347092
    in cache 3 are present video10, video1, video5, video2, video4, video24, video16
    in cache 2 are present video1, video31, video8, video5
    in cache 9 are present video5, video23, video43, video4, video0, video16
    in cache 5 are present video13, video10, video8, video0, video81
    in cache 0 are present video15, video10, video5, video0, video82, video16, video46
    in cache 1 are present video13, video6, video99, video7, video1, video65, video16
    in cache 8 are present video10, video2, video3, video0
    in cache 6 are present video30, video1, video5, video27, video4, video0, video16
    in cache 7 are present video54, video74, video1, video21, video5
    in cache 4 are present video17, video32, video3, video4


It works! and only 130 seconds and 60mb to find a solution.

I tried the second dataset and after 5Gb of memory and half hour of computation the end was not in is sight, so it's pretty safe to say that this way of solving this problem is not particularly viable. Has i wrote at the start of this notebook, a dividi et impera solution, where only a small fraction of the requests are analyzed each time might produce good enough results.
