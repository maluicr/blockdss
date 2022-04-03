# blockdss

blockdss R package generates geostatistical disease maps using block direct sequential simulation algorithm.

It is designed to create disease rate maps with associated spatial uncertainty, but may also be used for similar problems such as mortality rates, or applied to other fields like ecology or criminology. Details on the geostatistical algorithm used in blockdss are described in Azevedo et al^[https://doi.org/10.1186/s12942-020-00221-5].

# R wrapper

blockdss R package is a wrapper for running dss.c.64.exe - a software tool for running geostatistical algorithms - and pixelate, an R package for creating geostatistical maps with varying pixel sizes to represent disease mapping and spatial uncertainty in a single-map.

This package provides convenient wrappers to write data frames in format readable by dss.c.64.exe (geoEAS), setting the required parameters, running block direct sequential simulation algorithm, reading the files generated by dss.c.64.exe, and pixelate spatially continuous predictions as per their average uncertainty.

# Getting started

1. download dss.c.64.exe from https://github.com/maluicr/dss (follow the download instructions)
2. install required packages using the following code (please accept any suggested package updates): 

```r
# install devtools from CRAN
if (!require("devtools")){
  install.packages("devtools")
  }
  
  # install packages from GitHub
  devtools::install_github("maluicr/blockdss", dependencies = TRUE)
  devtools::install_github("aimeertaylor/pixelate", build_vignettes = TRUE, dependencies = TRUE)
```

# Example

As an example, blockdss functions are used to map covid-19 incidence in Portugal on 15 January 2021. The covid dataset `ptdata` and the regular gridded `ptgrid` comes with the package `blockdss`. The coordinate reference system is ETRS89 / Portugal TM06 (EPSG: 3763) and coordinates are in metres. 

```r
# load required libraries
library(blockdss)
library(pixelate)
library(sp)
library(raster)
```

### Load data 

Disease data is a data frame with spatial data point locations, region id, number of disease cases and size of population. Gridded data refers to a rectangular grid with region id values at all simulation x,y coordinates (grid nodes). All id regions represented in disease data should be represented by 1 or more grid nodes.


```r
data(ptdata) # check ?ptdata
data(ptgrid) # check ?ptgrid
head(ptdata)
head(ptgrid)
```

We need to transform `ptgrid` to `SpatialPixelsDataFrame`:

```r
# set coordinate reference system
coordinates(ptgrid) <- ~x+y
proj4string(ptgrid) <- CRS("+init=epsg:3763")
class(ptgrid)
gridded(ptgrid) <- T
class(ptgrid)
```

### Incidence rates

You can produce incidence rates and variance-error term per region, with function `irates()`. 

```r
# compute rates and error variance
inc <- irates(dfobj = ptdata, oid = "oid_", xx = "x", yy = "y", zz = "t",
               cases = "ncases", pop = "pop19", casesNA = 1, day = "2021015")
```

An incidence file (.out) is created in a readable format for dss.c.64.exe and stored in './input' folder.

### Block file

A block file (.out) is required for block simulation with dss.c.64.exe. `blockfile()` function transforms the grid file into a readable format for dss.c.64.exe and saves it into a text file (.out) in './input' folder.

```r
# create block file
block <- blockfile(inc, ptgrid)

# plot grid input
plot(block$ingrid)
```

### Mask file

A mask file is also required for dss.c.64.exe. `maskfile()` function produces the required file.

```r
# create mask file
mask <- maskfile(block)
```

### Compute variogram

For now, only omnidirecional variograms are computed.

```r
# compute experimental variogram
vexp <- varexp(inc, lag = 7000, nlags = 25)

# plot experimental variogram
plot(vexp[["semivar"]][1:2], ylab = expression(paste(gamma, "(h)")), xlab = "h (in m)", main = "Semi-variogram") 

# add sill estimate
abline(h = vexp[["weightsvar"]], col ="red", lty = 2)
```

### Fit variogram model

For now, only Spherical and Exponential models are implemented.

```r
# compute theoretical variogram
vmod <- varmodel(vexp, mod = "sph", nug = 0, ran = 35000, sill = vexp[["weightsvar"]])

# add fitted variogram
lines(vmod[["fittedval"]]) 
```

Model parameters and shape are stored for dss.c.64.exe.

### Write dss.c.64.exe parameters file

Function `ssdpars()` generates a parameters file (.par), invokes dss.c.64.exe specified by the parameters file and generates simulated maps (.out).

```r
# run ssdir.par
ssdpars(blockobj = block, maskobj = mask, dfobj = inc, varmobj = vmod, 
        simulations = 5, radius1 = 35000, radius2 = 35000)
```

Note that this process may take a while, depending mostly on the number of simulation nodes and number of simulations specified.

### Median E-type, uncertatinty and pixelated map

```r
# export .out maps to raster
maps <- outraster(block, emaps = T)

# plot simulations
spplot(maps[["simulations"]])

# plot etype and uncertainty
spmap(maps, mapvar = "etype", legname = "Median \n incidence")
spmap(maps, mapvar = "uncertainty", legname = "IQ range")

# plot pixelated map
pxmap(maps)
```




