# Overlap Correction with Linear Models
date: 2021-04-29
----

!!! note
This is a tutorial, not so much a documentation.

```@setup index
using Plots; gr()
Plots.reset_defaults();
```
## Installation
First we have to install some packages. in julia you would do this either by putting a `]` in the REPL ("julia-commandline").
This should result in
`(Unfold) pkg> ` - but if you see `(@v1.6) pkg> `  instead, you still have to activate your environment (using `cd("/path/to/your/project")` and `]activate .` or `]activate /path/to/your/project/`)

note that you should have done this already to install Unfold in the first place. have a look at the Readme.md - there we use the Pkg.add("") syntax, which is equivalent to the `]` package manager.
Now we are ready to add packages:

`add StatsModels,MixedModels,DataFrames,DSP.conv,Plots`

Next we have to make sure to be in the `Unfold/docs` folder, else the tutorial will not be able to find the data. Thus `cd("./docs")` in case you cd'ed already to the Unfold project.


## Setting up & loading the data
```@example Main

using StatsModels, MixedModels, DataFrames
import DSP.conv
import Plots
using Unfold
include("../../test/test_utilities.jl"); # to load the simulated data
```







In this notebook we will fit regression models to (simulated) EEG data. We will see that we need some type of overlap correction, as the events are close in time to each other, so that the respective brain responses overlap.
If you want more detailed introduction to this topic check out [our paper](https://peerj.com/articles/7838/)
```@example Main
data, evts = loadtestdata("testCase2",dataPath="../../test/data/");
```

The data has little noise and the underlying signal is a pos-neg spike pattern
```@example Main
Plots.plot(range(1/50,length=200,step=1/50),data[1:200])
Plots.vline!(evts[evts.latency.<=200,:latency]./50) # show events
```

Now have a look at the events
```@example Main
show(first(evts,6,),allcols=true)
```



## Traditional Mass Univariate Analysis
In order to demonstrate why overlap correction is important, we will first epoch the data and fit a linear model to each time point.
This is a "traditional mass-univariate analysis".
```@example Main
# we have multi channel support
data_r = reshape(data,(1,:))
# cut the data into epochs
data_epochs,times = Unfold.epoch(data=data_r,tbl=evts,τ=(-0.4,0.8),sfreq=50);
```
!!! note
In julia, `missing` is supported throughout the ecosystem. Thus, we can have partial trials and they will be incorporated / ignored at the respective functions.




We define a formula that we want to apply to each point in time
```@example Main
f  = @formula 0~1+conditionA+conditionB # 0 as a dummy, we will combine wit data later
```






We fit the `UnfoldLinearModel` to the data
```@example Main
m,results = Unfold.fit(UnfoldLinearModel,f,evts,data_epochs,times); nothing #hide
```



The object has the following fields
```@example Main
m
```


Which contain the model, the original formula, the original events and returns extra a *tidy*-dataframe with the results
```@example Main


first(results,6)
```


We can also plot it:
```@example Main
Plots.plot(results.colname_basis,results.estimate,
        group=results.term,
        layout=1,legend=:outerbottom)
```




!!! note
(`:colname_basis` is used instead of `:time` [this might change]. The reason is that not all basisfunctions have a time dimension)


As can be seen a lot is going on here. As we will see later, most of the activity is due to overlap with the next event


## Basis Functions
#### HRF / BOLD
We are now ready to define a basisfunction. There are currently only few basisfunction implemented.
We first have a look at the BOLD-HRF basisfunction:

```@example Main


TR = 1.5
bold = hrfbasis(TR) # using default SPM parameters
eventonset = 1.3
Plots.plot(bold.kernel(eventonset))
```



Classically, we would convolve this HRF function with a impulse-vector, with impulse at the event onsets
```@example Main


y = zeros(100)
y[[10,30,37,45]] .=1
y_conv = conv(y,bold.kernel(0))
Plots.plot(y_conv)
```

Which one would use as a regressor against the recorded BOLD timecourse.

note that events could fall inbetween TR (the sampling rate). Some packages subsample the time signal, but in `Unfold` we can directly call the `bold.kernel` function at a given event-time, which allows for non-TR-multiples to be used.

### FIR Basis Function

Okay, let's have a look at a different basis function: The FIR basisfunction.

```@example Main


basisfunction = firbasis(τ=(-0.4,.8),sfreq=50,name="myFIRbasis")
Plots.plot(basisfunction.kernel(0))
```



Not very clear, better show it in 2D
```@example Main


basisfunction.kernel(0)[1:10,1:10]
```
(all `.` are `0`'s)



The FIR basisset consists of multiple basisfunctions. That is, each event will now be *timeexpanded* to multiple predictors, each with a certain time-delay to the event onset.
This allows to model any arbitrary linear overlap shape, and doesn't force us to make assumptions on the convolution kernel (like we had to do in the BOLD case)


## Timeexpanded / Deconvolved ModelFit
Remember our formula from above
```@example Main


f
```





For the left-handside we use "0" as the data is separated from the events. This will in the future allow us to fit multiple channels easily.

And fit a `UnfoldLinearModel`. Not that instead of `times` as in the mass-univariate case, we have a `BasisFunction` object now.
```@example Main


m,results = fit(UnfoldLinearModel,Dict(Any=>(f,basisfunction)),evts,data); nothing #hide
```

!!! note
in `(Any=>(f,basisfunction)`, the `Any` means to use all rows in `evts`. In case you have multiple events, you'd want to specify multiple basisfunctions e.g. 
```
Dict("stimulus"=>(f1,basisfunction1),
 "response"=>(f2,basisfunction2))
```
You likely have to specify a further argument to `fit`: `eventcolumn="type"` with `type` being the column in `evts` that codes for the event (stimulus / response in this case)



```@example Main


Plots.plot(results.colname_basis,results.estimate,
        group=results.term,
        layout=1,legend=:outerbottom)
```




Cool! All overlapping activity has been removed and we recovered the simulated underlying signal.



