# Dynamical Factor Analysis and Multispecies clustering

## Reproducible example

### Load packages and functions

```{r}

# Load packages

source("package_used.R")

# Load functions for time-series
source("function_ts.R")

# Load functions for Dynamic Factor Analysis (DFA) with TMB
source('function_dfa_clean.R')

```

### Reproducible data

```{r}
seed_id <- 0
id_vec <- c()
n_sp_init <- 15
nb_group_exp <-2
cum_perc <- c(10,5)
n_y <- 20
n_lt <- 4

y_init <- data.frame(t(rep(NA,n_y))) # latent trends
for(i in 1:n_lt){
    set.seed(i+10)
    y_ts <- c()
    y_ts[1] <- rnorm(n = 1, mean = 0, sd = 1)
    for (t in 2:n_y) {
        r.w <- rnorm(n = 1, mean = 0, sd = 1)
        y_ts[t] <- y_ts[t - 1] + r.w
    }
    y_ts <- y_ts + abs(min(y_ts))+1
    y_ts <- exp(scale(log(y_ts)))
    #max_new <- max(y_ts) - mean(y_ts)/4
    #min_new <- min(y_ts) + mean(y_ts)/4
    #y_ts <- scales::rescale(y_ts, to=c(min_new, max_new))
    y_init[i,] <- y_ts
}

for(g in 1:nb_group_exp){
    nb_sp_g <- cum_perc[g]
    assign(paste0("nb_sp_g",g),nb_sp_g)
    id_vec <- c(id_vec,rep(g,nb_sp_g))
    for(lt in 1:n_sp_init){
        seed_id <- seed_id + 1
        set.seed(seed_id)
        mean_u_g <- runif(1, -1, 1)
        lf_u_g <- rnorm(nb_sp_g, mean_u_g, 0.1)
        assign(paste0("mean_u",lt,"_g",g),mean_u_g) # mean of loading factors in group g for latend trend lt
        assign(paste0("lf_u",lt,"_g",g),lf_u_g) # loading factors for each ts of group g for latend trend lt

    }
}
id_vec <- id_vec[1:n_sp_init]

y <- data.frame(t(rep(NA,(n_y+2))))
obs_se <- data.frame(t(rep(NA,(n_y+1))))

for(i in 1:n_sp_init){ # get simulated ts from loadings
    set.seed(i)
    noise <- rnorm(n_y,0,0.01)
    y[i,1] <- obs_se[i,1] <- sprintf("SP%03d",i)
    y_ts <- rep(0,n_y)
    g <- id_vec[i]
    i_g <- which(which(id_vec==g)==i) # new index for i in group g
    for(lt in 1:n_lt){
        lf_u_g <- get(paste0("lf_u",lt,"_g",g))
        y_ts <- y_ts + as.numeric(y_init[lt,])*lf_u_g[i_g]
    }
    y_ts <- y_ts + noise
    y_ts <- y_ts + abs(min(y_ts)) + 1
    y_ts <- exp(scale(log(y_ts)))
    y[i,2:(n_y+1)] <- y_ts
    y[i,(n_y+2)] <- id_vec[i]
    obs_se[i,2:(n_y+1)] <- abs(rnorm(n_y,0,0.1))
    obs_se[obs_se>1] <- 1
}  


y_ex <- data.table(y[,1:(n_y+1)])
obs_se_ex <- data.table(obs_se)
names(y_ex) <- names(obs_se_ex) <- c("code_sp",1995:(1995+n_y-1))
y_ex$code_sp <- obs_se_ex$code_sp <- sprintf("SP%03d",1:nrow(y_ex))
species_ex <- data.frame(name_long=sprintf("species %03d",1:nrow(y_ex)), code_sp=y_ex$code_sp)

```

### Data specification

Three datasets should be given as input for the main function 'make_dfa':

- 'data_ts' contains species time-series and should be provided as a 'data.table' with species time-series in row and the first column for species names'codes and years as column names :

```{r}

data_ts <- y_ex

head(data_ts)

```

- 'data_ts_se' contains observation errors of species time-series. It should be provided as a 'data.table' with species time-series in row and the first column for species names'codes and years as column names:

```{r}

data_ts_se <- obs_se_ex

head(data_ts_se)

```

'species_sub' contains species names. It should be provided as a 'data.frame' with species in row, the first column for complete species names and the second column for species names'codes:

```{r}

species_sub <- species_ex

head(species_sub)

```

### Run the analysis

```{r}

ex_dfa_clust <- make_dfa(data_ts = data_ts, # Dataset of time series
data_ts_se = data_ts_se, # Dataset of observation error of time series 
species_sub = species_sub, # Species names
# optional variables
nfac=0, # Number of trends for the DFA, 0 to estimate the best number of trends
mintrend=1, # Minimum number of trends to test
maxtrend=5, # Maximum number of trends to test
AIC=TRUE, # Display AIC
nboot=500, # Number of bootstrap for clustering
silent = TRUE, # Silence optimisation
control = list()) # Specify changes for DFA control options

```

### Display results

```{r}

# Species time-series

head(ex_dfa_clust[[1]]) # code_sp: species names'code; Year: year; value_orig: input values for species time-series; se_orig: input values for observation error of species time-series; value: back transformed values for species time-series (should be identical to value_orig); se.value:  back transformed values for species time-series (should be identical to value_orig); pred.value: predictided values for species time-series from DFA, pred_se.value: predictided values for standard error of species time-series from DFA; name_long: species names.

# DFA latent trends

head(ex_dfa_clust[[2]]) # Year: year; variable: latent trend id; value: latent trend values; se.value: standard error of latent trends; rot_tr.value: rotated values for latent trends.

# DFA loading factors

head(ex_dfa_clust[[3]]) # code_sp: species names'code; variable: latent trend id; value: loading factors; se.value: standard error of loading factors; name_long: species names.

# Plot species time-series

ex_dfa_clust[[4]]

# Plot latent trends

ex_dfa_clust[[5]]

# Plot loading factors

ex_dfa_clust[[6]]

# Plot species clusters

ex_dfa_clust[[7]][[1]]

# Plot time-series of cluster barycentres

ex_dfa_clust[[7]][[2]]$g1
ex_dfa_clust[[7]][[2]]$g2

# Akaike information criterion (AIC) of DFA

ex_dfa_clust$aic # or ex_dfa_clust[[8]]

# Optimisation output of DFA

head(ex_dfa_clust$sdRep) # or head(ex_dfa_clust[[9]])

# Results of species clustering

head(ex_dfa_clust$group[[1]][[1]]) # code_sp: species names'code; PC1: coordinate on PCA first axis; PC2: coordinate on PCA second axis; group: cluster id; Xn: rotated loading factors for each latent trend n; uncert: species stability into its cluster; name_long: species names.

# Cluster barycentres

ex_dfa_clust$group[[1]][[2]]

# Variance captured by PCA first two axes

ex_dfa_clust$group[[1]][[3]]

# Cluster position in the first factorial plan

ex_dfa_clust$group[[2]]

# Cluster stability

ex_dfa_clust$group[[3]]

# Cluster dispersion

ex_dfa_clust$group[[4]]

# Time-series of cluster barycentres

head(ex_dfa_clust[[11]])


```


## Empirical data

### Load and prepare data

```{r}
# Download extract and load data from https://www.gbif.org/occurrence/download?dataset_key=91fa1a0d-a208-40aa-8a6e-f2c0beb9b253 (an free account is necessary) (Type: Darwin Core Archive)

bird_se_raw <- read.csv("raw_data/occurrence.txt", header = T, sep="\t")

# Cleaning data

bird_se_clean <- bird_se_raw[bird_se_raw$class=="Aves",c("class","order","family", "genus","species",
                                                            "specificEpithet","infraspecificEpithet","taxonRank",
                                                            "organismQuantity","decimalLatitude","decimalLongitude",
                                                            "day","month","year","taxonKey","speciesKey","countryCode",
                                                            "level1Gid","level2Gid","iucnRedListCategory")]

# add a code by species from species name

species_data <- data.frame(name_long = unique(bird_se_clean$species[bird_se_clean$taxonRank != "GENUS"]))
species_data$code_sp <- paste0(toupper(substr(species_data$name_long, 1, 3)),
                               toupper(substr(sub(".* ", "", species_data$name_long), 1, 3)))
species_data$code_sp[species_data$name_long=="Corvus corax"] <- "CORCOX"
species_data$code_sp[species_data$name_long=="Phylloscopus trochilus"] <- "PHYTRU"
species_data$code_sp[species_data$name_long=="Sterna paradisaea"] <- "STEPAD"
species_data$genus <- sub(" .*", "", species_data$name_long)
species_data$species <- sub(".* ", "", species_data$name_long)

bird_se_clean <- merge(bird_se_clean, species_data[,c("code_sp", "name_long")],
                       by.x = c("species"), by.y = c("name_long"), all.x = T)

species_data$class <- bird_se_clean$class[match(species_data$code_sp, bird_se_clean$code_sp)]
species_data$order <- bird_se_clean$order[match(species_data$code_sp, bird_se_clean$code_sp)]
species_data$family <- bird_se_clean$family[match(species_data$code_sp, bird_se_clean$code_sp)]
species_data$iucnRedListCategory <- bird_se_clean$iucnRedListCategory[match(species_data$code_sp, bird_se_clean$code_sp)]



# create route number from coordinate

route_data <- paste0(bird_se_clean$decimalLatitude, sep="_", bird_se_clean$decimalLongitude)

require(stringr)
route_data <- data.frame(code_route = paste0("R",str_pad(1:length(unique(route_data)), 3, pad = "0")),
                         coordinate_chr = unique(route_data))
route_data$lat <- as.numeric(sub("_.*", "", route_data$coordinate_chr))
route_data$lon <- as.numeric(sub(".*_", "", route_data$coordinate_chr))

# view route on map, need to know the projection

require(sf)
require(rnaturalearth)
require(ggplot2)
require(sp)
require(rgdal)

worldmap <- ne_countries(scale = 'medium', type = 'countries',returnclass = 'sf')
sweden_map_wgs84 <- worldmap[worldmap$sovereign=="Sweden",]
sweden_map_moll <- sf::st_transform(sweden_map_wgs84, "+proj=moll")
sweden_map_swe <- sf::st_transform(sweden_map_wgs84, "+init=epsg:3006")

route_data_coord <- route_data
coordinates(route_data_coord) <- ~lon+lat
proj4string(route_data_coord) <- CRS("+proj=longlat +datum=WGS84")
route_data_coord <- spTransform(route_data_coord, CRSobj = "+proj=moll")
route_data_moll <- as.data.frame(coordinates(route_data_coord))
route_data_moll$code_route <- route_data_coord$code_route
route_data_coord <- spTransform(route_data_coord, CRSobj = "+init=epsg:3006")
route_data_swe <- as.data.frame(coordinates(route_data_coord))
route_data_swe$code_route <- route_data_coord$code_route

ggplot() + geom_sf(data=sweden_map_swe) +
  geom_tile(data = route_data_swe, aes(x=lon, y=lat), width=25000, height=25000, alpha=0.5) +
  theme_void() + coord_sf(datum=NA)

route_data <- merge(route_data, route_data_moll, by="code_route", all=T)
route_data <- merge(route_data, route_data_swe, by="code_route", all=T)
names(route_data)[3:8] <- c("lat_wgs", "lon_wgs", "lon_moll", "lat_moll", "lon_swe", "lat_swe")
route_data$coordinate_chr <- NULL

# associate routes with the main dataset

bird_se_clean <- merge(bird_se_clean, route_data[,c("code_route", "lat_wgs", "lon_wgs")],
                       by.x = c("decimalLatitude", "decimalLongitude"), by.y = c("lat_wgs", "lon_wgs"), all.x = T)

route_data$level1Gid <- bird_se_clean$level1Gid[match(route_data$code_route, bird_se_clean$code_route)]
route_data$level2Gid <- bird_se_clean$level2Gid[match(route_data$code_route, bird_se_clean$code_route)]


# associate route and region

# require(remotes)
# remotes::install_github("filipwastberg/swemaps2")
require(swemaps2)

municip_data <- municipality
municip_data <- sf::st_transform(municip_data, "+init=epsg:3006")
municip_data <- as(municip_data, 'Spatial')
municip_data$area_sqkm <- gArea(municip_data, byid = T)/1000000

route_data_sp <- SpatialPoints(route_data[,c("lon_swe","lat_swe")],
                               proj4string = CRS("+init=epsg:3006"))
route_data <- data.frame(route_data, over(route_data_sp, municip_data))

equival <- data.frame(ln_kod2 = as.character(levels(as.factor(route_data$ln_kod))),
                      level1Gid = as.character(apply(table(route_data$level1Gid,route_data$ln_kod),
                                                     2,function(x){names(which.max(x))})))
route_data <- merge(route_data,equival, by="level1Gid", all.x=T)
route_data$ln_kod[is.na(route_data$ln_kod)] <- route_data$ln_kod2[is.na(route_data$ln_kod)]

# ecoregion from https://doi.org/10.1016/j.landurbplan.2020.103838

equival2 <- data.frame(ln_kod = as.character(levels(as.factor(route_data$ln_kod))),
                       ecoreg = c(rep("south",12),"north",rep("south",2),rep("north",6)),
                       subecoreg = c(rep("hemiboreal",8),rep("nemoral",3),"hemiboreal",
                                     "s_boreal",rep("hemiboreal",2),rep("s_boreal",4),
                                     rep("n_boreal",2)))

route_data <- merge(route_data,equival2, by="ln_kod", all.x=T)

# check
require(viridis)
ggplot() + geom_sf(data=sweden_map_swe) +
  geom_tile(data = route_data, aes(x=lon_swe, y=lat_swe, fill=ln_kod),
            width=25000, height=25000, alpha=0.5) + scale_fill_viridis(discrete = T) +
  theme_void() + coord_sf(datum=NA)
ggplot() + geom_sf(data=sweden_map_swe) +
  geom_tile(data = route_data, aes(x=lon_swe, y=lat_swe, fill=ecoreg),
            width=25000, height=25000, alpha=0.5) + scale_fill_viridis(discrete = T) +
  theme_void() + coord_sf(datum=NA)
ggplot() + geom_sf(data=sweden_map_swe) +
  geom_tile(data = route_data, aes(x=lon_swe, y=lat_swe, fill=subecoreg),
            width=25000, height=25000, alpha=0.5) + scale_fill_viridis(discrete = T) +
  theme_void() + coord_sf(datum=NA)

# incorporate 0 in the data (species not present while the route is sample this given year)

require(maditr)
bird_se_clean_tot <- dcast(bird_se_clean, countryCode+code_route+year~code_sp,
                           fun.aggregate = sum, value.var="organismQuantity")

bird_se <- melt(bird_se_clean_tot, id.vars = c("countryCode", "code_route", "year"))
names(bird_se)[4:5] <- c("code_sp", "abund")
bird_se$code_sp <- as.character(bird_se$code_sp)
bird_se <- bird_se[bird_se$code_sp!="NA",]

bird_se <- merge(bird_se, route_data, by="code_route", all.x = T)
bird_se$order <- species_data$order[match(bird_se$code_sp, species_data$code_sp)]
bird_se$family <- species_data$family[match(bird_se$code_sp, species_data$code_sp)]
bird_se$genus <- species_data$genus[match(bird_se$code_sp, species_data$code_sp)]
bird_se$species <- species_data$species[match(bird_se$code_sp, species_data$code_sp)]
bird_se$name_long <- species_data$name_long[match(bird_se$code_sp, species_data$code_sp)]
bird_se$iucnRedListCategory <- species_data$iucnRedListCategory[match(bird_se$code_sp, species_data$code_sp)]

```


### Compute species time-series

```{r}
# look at abundance for each species

bird_se <- readRDS("output/bird_se.rds")

# Use data from 1998 because low number of routes monitored in 1996 and 1997: https://doi.org/10.34080/os.v17.22684 )

bird_se_1998 <- droplevels(bird_se[bird_se$year>1997,])
saveRDS(bird_se_1998,"output/bird_se_1998.rds")

ab_sp <- as.data.frame(bird_se_1998 %>% group_by(code_sp) %>% summarize(ab_tot=sum(abund)))

ggplot(ab_sp, aes(x=reorder(code_sp, -ab_tot, sum), y=ab_tot)) + 
  geom_bar(stat = "identity")+ theme_classic() + scale_y_continuous(trans = "log")+
  theme(axis.text.x = element_text(angle = 90, vjust = 0.5, hjust=1)) + xlab("Species")+
  ylab("Abundance (log)")

ab_sp2 <- ab_sp[order(ab_sp$ab_tot, decreasing = T),]
ab_sp2$perc_cum <- cumsum(ab_sp2$ab_tot)/sum(ab_sp2$ab_tot)
saveRDS(ab_sp2,"output/ab_sp2.rds")

# time-series from the SBBS

## example for one species

data_ex <- droplevels(bird_se_1998[bird_se_1998$code_sp=="CORFRU",])
timestep <- length(levels(as.factor(data_ex$year)))-1

glm1 <- glm(abund~as.factor(code_route)+as.factor(year), data=data_ex, family = quasipoisson)
sglm1 <- summary(glm1)

## as link function is log, estimates need to be back transformed

coef_yr <- tail(matrix(sglm1$coefficients[,1]), timestep)
coef_yr <- rbind(1, exp(coef_yr))

error_yr <- tail(matrix(sglm1$coefficients[,2]), timestep)
error_yr <- rbind(0, error_yr)

## CIs

require(arm)
glm1.sim <- sim(glm1)
lc_inf_sim <- c(1, exp(tail(apply(coef(glm1.sim),2, quantile, .025), timestep)))
lc_sup_sim <- c(1, exp(tail(apply(coef(glm1.sim),2, quantile, .975), timestep)))

## plot

require(see)

tab_plot <- data.frame(year=as.numeric(as.character(levels(as.factor(data_ex$year)))),
                       value=coef_yr, error=error_yr, LL=lc_inf_sim, UL=lc_sup_sim)
ggplot(tab_plot, aes(year, value)) + geom_line() +
  geom_line(aes(y=LL), linetype="dashed") +
  geom_line(aes(y=UL), linetype="dashed") +
  ylab("Relative abundance") + xlab("Years") +
  theme_modern()

## for all species

### for loop from French birds analysis

listSp<-levels(as.factor(bird_se_1998$code_sp))
  
i<-0
for (sp in listSp) {
    print(i)
  
    i <- i + 1
    
    ## d data for species i
    sp<-listSp[i]
    d <- droplevels(bird_se_1998[bird_se_1998$code_sp==sp,])
    
    ## Occurrence
    ## number of route followed by year
    nb_route <- tapply(rep(1,nrow(d)),d$year,sum)
    ## number of route with species i by year
    nb_route_presence <- tapply(ifelse(d$abund>0,1,0),d$year,sum)
    year <- as.numeric(as.character(levels(as.factor(d$year))))
    firstY <- min(year)
    lastY <- max(year)
    timestep <- length(year)-1
    
    ## table for analysis result
    threshold_occurrence <- 3
    tab_ana <- data.frame(year=rep(year,2),val=c(nb_route,nb_route_presence),LL = NA,UL=NA,
                       catPoint=NA,pval=NA,
                       curve=rep(c("route","presence"),each=length(year)))
    tab_ana$catPoint <- ifelse(tab_ana$val == 0,"0", ifelse(tab_ana$val < threshold_occurrence,
                                                           "inf_threshold",NA))
    
    ## raw abundance
    ## abundance by year
    abund <- tapply(d$abund,d$year,sum)
    ## table for figure
    threshold_abundance <- 5
    tab_fig <- data.frame(year=year,val=abund,LL = NA,UL=NA,catPoint=NA,pval=NA)
    tab_fig$catPoint <- ifelse(tab_fig$val == 0,"0",ifelse(tab_fig$val < threshold_abundance,
                                                           "inf_threshold",NA))
    
    # remove criteria
    remove_sp <- FALSE
    
    ## if first year empty
    if(tab_fig$val[1]==0){remove_sp <- TRUE}
    
    ## if four consecutive years empty
    ab_vec <- paste(tab_fig$val,collapse="")
    if(str_detect(ab_vec, "0000")){remove_sp <- TRUE}
    
    ## if less than consecutive years
    ab_vec2 <- paste(sign(tab_fig$val),collapse="")
    if(!str_detect(ab_vec2, "111")){remove_sp <- TRUE}
    
    if(anyNA(tab_fig$catPoint) & anyNA(tab_ana$catPoint[tab_ana$curve=="presence"]) & remove_sp==F){
      
      ## GLM abundance variation
      glm1 <- glm(abund~as.factor(code_route)+as.factor(year),data=d,family=quasipoisson)
      sglm1 <- summary(glm1)
      
      ## as link function is log, estimates need to be back transformed
      coef_yr <- tail(matrix(sglm1$coefficients[,1]), timestep)
      coef_yr <- rbind(1, exp(coef_yr))
      error_yr <- tail(matrix(sglm1$coefficients[,2]), timestep)
      error_yr <- rbind(0, error_yr)
      pval <- c(1,tail(matrix(coefficients(sglm1)[,4]),timestep))
      
      ## CIs
      glm1.sim <- sim(glm1)
      ci_inf_sim <- c(1, exp(tail(apply(coef(glm1.sim),2, quantile, .025), timestep)))
      ci_sup_sim <- c(1, exp(tail(apply(coef(glm1.sim),2, quantile, .975), timestep)))

      ## table for result and figures
      thresold_signif <- 0.05
      tab_res <- data.frame(year,val=coef_yr,
                         LL=ci_inf_sim,UL=ci_sup_sim,
                         catPoint=ifelse(pval<thresold_signif,"significatif",NA),pval)
      ## cleaning out of range CIs			   
      tab_res$UL <- ifelse(nb_route_presence==0,NA,tab_res$UL)
      tab_res$UL <-  ifelse(tab_res$UL == Inf, NA,tab_res$UL)
      tab_res$UL <-  ifelse(tab_res$UL > 1.000000e+20, NA,tab_res$UL)
      tab_res$UL[1] <- 1
      tab_res$val <-  ifelse(tab_res$val > 1.000000e+20,1.000000e+20,tab_res$val)
      
      ## overdispersion index
      dispAn <- sglm1$deviance/sglm1$null.deviance
      
      ## class uncertainity
      if(dispAn > 2 | (median(nb_route_presence)<threshold_occurrence & median(abund)<threshold_abundance)) catIncert <- "Uncertain" else catIncert <-"Good"
      vecLib <-  NULL
      if(dispAn > 2 | median(nb_route_presence)<threshold_occurrence) {
        if(median(nb_route_presence)<threshold_occurrence) {
          vecLib <- c(vecLib,"too rare species")
        }
        if(dispAn > 2) {
          vecLib <- c(vecLib,"deviance")
        }
      }
      reason_uncert <-  paste(vecLib,collapse=" and ")
      
      ## table for saving results      
      tab_tot <- data.frame(code_sp=sp, year = tab_res$year, nb_year=timestep,
                          firstY=firstY, lastY=lastY,
                          relative_abundance=round(tab_res$val,3),
                          CI_inf = round(tab_res$LL,3), CI_sup = round(tab_res$UL,3),
                          Standard_error = round(error_yr,4),
                          p_value = round(tab_res$pval,3), signif = !is.na(tab_res$catPoint),
                          nb_route,nb_route_presence,abundance=abund,
                          mediane_occurrence=median(nb_route_presence), mediane_ab=median(abund) ,
                          valid = catIncert, uncertanity_reason = reason_uncert)
      
    }
    else{
      tab_tot <- data.frame(code_sp=sp, year = tab_res$year, nb_year=timestep,
                            firstY=firstY, lastY=lastY,
                            relative_abundance=NA,
                            CI_inf = NA, CI_sup = NA,
                            Standard_error = NA,
                            p_value = NA, signif = NA,
                            nb_route,nb_route_presence,abundance=abund,
                            mediane_occurrence=median(nb_route_presence), mediane_ab=median(abund) ,
                            valid = NA, uncertanity_reason = NA)
    }
    
    
    if(sp==listSp[1]) {
      glm_res <- tab_tot
    } else  {
      glm_res <- rbind(glm_res,tab_tot)
    }
} 

# compare with https://www.fageltaxering.lu.se/resultat/trender/allatrendertillsammans
ggplot(glm_res[glm_res$code_sp==levels(as.factor(glm_res$code_sp))[1],], aes(year, relative_abundance)) +
  geom_line() + geom_line(aes(y=CI_inf), linetype="dashed") + geom_line(aes(y=CI_sup), linetype="dashed") +
  ylab("Relative abundance") + xlab("Years") + theme_modern()


# function for Swedish data

get_ts <- function(data_bird_input){
  
    ## d data for species i
    d <- droplevels(data_bird_input)
    sp <- levels(as.factor(d$code_sp))
    
    ## Occurrence
    ## number of route followed by year
    nb_route <- tapply(rep(1,nrow(d)),d$year,sum)
    ## number of route with species i by year
    nb_route_presence <- tapply(ifelse(d$abund>0,1,0),d$year,sum)
    year<-as.numeric(as.character(levels(as.factor(d$year))))
    firstY <- min(year)
    lastY <- max(year)
    timestep <- length(year)-1
    
    ## table for analysis result
    threshold_occurrence <- 3
    tab_ana <- data.frame(year=rep(year,2),val=c(nb_route,nb_route_presence),LL = NA,UL=NA,
                          catPoint=NA,pval=NA,
                          curve=rep(c("route","presence"),each=length(year)))
    tab_ana$catPoint <- ifelse(tab_ana$val == 0,"0", ifelse(tab_ana$val < threshold_occurrence,
                                                            "inf_threshold",NA))
    
    ## raw abundance
    ## abundance by year
    abund <- tapply(d$abund,d$year,sum)
    ## table for figure
    threshold_abundance <- 5
    tab_fig <- data.frame(year=year,val=abund,LL = NA,UL=NA,catPoint=NA,pval=NA)
    tab_fig$catPoint <- ifelse(tab_fig$val == 0,"0",ifelse(tab_fig$val < threshold_abundance,
                                                           "inf_threshold",NA))
    
    # remove criteria
    remove_sp <- FALSE
    
    ## if first year empty
    if(tab_fig$val[1]==0){remove_sp <- TRUE}
    
    ## if four consecutive years empty
    ab_vec <- paste(tab_fig$val,collapse="")
    if(str_detect(ab_vec, "0000")){remove_sp <- TRUE}
    
    ## if less than consecutive years
    ab_vec2 <- paste(sign(tab_fig$val),collapse="")
    if(!str_detect(ab_vec2, "111")){remove_sp <- TRUE}
    
    if(anyNA(tab_fig$catPoint) & anyNA(tab_ana$catPoint[tab_ana$curve=="presence"]) & remove_sp==F){
      
      ## GLM abundance variation
      glm1 <- glm(abund~as.factor(code_route)+as.factor(year),data=d,family=quasipoisson)
      sglm1 <- summary(glm1)
      
      ## mean-centered values
      con.mat <- diag(length(year)) - 1/length(year)
      colnames(con.mat) <- year#firstY:lastY
      rg <- ref_grid(glm1, nuisance = 'code_route')
      sglm2 <- summary(contrast(rg, as.data.frame(con.mat)))
      
      ## as link function is log, estimates need to be back transformed
      # from sglm1 (first year set to 1 and se to 0)
      coef_yr <- tail(matrix(sglm1$coefficients[,1]), timestep)
      coef_yr <- rbind(1, exp(coef_yr))
      error_yr <- tail(matrix(sglm1$coefficients[,2]), timestep)
      error_yr <- rbind(0, error_yr)*coef_yr # approximated se values
      log_error_yr <- tail(matrix(sglm1$coefficients[,2]), timestep)
      log_error_yr <- rbind(0, log_error_yr)
      pval <- c(1,tail(matrix(coefficients(sglm1)[,4]),timestep))
      
      # from sglm2 (mean value to 0)
      coef_yr_m0 <- exp(sglm2$estimate)
      error_yr_m0 <- sglm2$SE*coef_yr_m0 # approximated se values
      log_error_yr_m0 <- sglm2$SE
      pval_m0 <- sglm2$p.value
      
      ## CIs
      glm1.sim <- sim(glm1)
      ci_inf_sim <- c(1, exp(tail(apply(coef(glm1.sim),2, quantile, .025), timestep)))
      ci_sup_sim <- c(1, exp(tail(apply(coef(glm1.sim),2, quantile, .975), timestep)))
      
      ## table for result and figures
      thresold_signif <- 0.05
      tab_res <- data.frame(year,val=coef_yr,val_m0=coef_yr_m0,
                            LL=ci_inf_sim,UL=ci_sup_sim,
                            catPoint=ifelse(pval<thresold_signif,"significatif",NA),pval)
      ## cleaning out of range CIs			   
      tab_res$UL <- ifelse(nb_route_presence==0,NA,tab_res$UL)
      tab_res$UL <-  ifelse(tab_res$UL == Inf, NA,tab_res$UL)
      tab_res$UL <-  ifelse(tab_res$UL > 1.000000e+20, NA,tab_res$UL)
      tab_res$UL[1] <- 1
      tab_res$val <-  ifelse(tab_res$val > 1.000000e+20,1.000000e+20,tab_res$val)
      tab_res$val_m0 <-  ifelse(tab_res$val_m0 > 1.000000e+20,1.000000e+20,tab_res$val_m0)
      
      ## overdispersion index
      dispAn <- sglm1$deviance/sglm1$null.deviance
      
      ## class uncertainity
      if(dispAn > 2 | (median(nb_route_presence)<threshold_occurrence & median(abund)<threshold_abundance)) catIncert <- "Uncertain" else catIncert <-"Good"
      vecLib <-  NULL
      if(dispAn > 2 | median(nb_route_presence)<threshold_occurrence) {
        
        if(median(nb_route_presence)<threshold_occurrence) {
          vecLib <- c(vecLib,"too rare species")
        }
        if(dispAn > 2) {
          vecLib <- c(vecLib,"deviance")
        }
      }
      reason_uncert <-  paste(vecLib,collapse=" and ")
      
      ## table for saving results      
      tab_tot <- data.frame(code_sp=sp, year = tab_res$year, nb_year=timestep,
                            firstY = firstY, lastY = lastY,
                            relative_abundance = tab_res$val,
                            CI_inf = tab_res$LL, CI_sup = tab_res$UL,
                            Standard_error = error_yr,
                            Log_SE = log_error_yr,
                            p_value = tab_res$pval,
                            relative_abundance_m0 = tab_res$val_m0,
                            Standard_error_m0 = error_yr_m0,
                            Log_SE_m0 = log_error_yr_m0,
                            p_value_m0 = pval_m0, signif = !is.na(tab_res$catPoint),
                            nb_route,nb_route_presence,abundance=abund,
                            mediane_occurrence=median(nb_route_presence), mediane_ab=median(abund) ,
                            valid = catIncert, uncertanity_reason = reason_uncert)
      
    }
    else{
      tab_tot <- data.frame(code_sp=sp, year = year, nb_year=timestep,
                            firstY=firstY, lastY=lastY,
                            relative_abundance=NA,
                            CI_inf = NA, CI_sup = NA,
                            Standard_error = NA,
                            p_value = NA, 
                            relative_abundance_m0 = NA,
                            Standard_error_m0 = NA,
                            Log_SE_m0 = NA,
                            p_value_m0 = NA,signif = NA,
                            nb_route,nb_route_presence,abundance=abund,
                            mediane_occurrence=median(nb_route_presence), mediane_ab=median(abund) ,
                            valid = NA, uncertanity_reason = NA)
    }
    
    return(tab_tot)
} 

# require(plyr) but need to be done before loading dplyr, see package script

ts_test <- ddply(droplevels(bird_se_1998[bird_se_1998$code_sp %in% levels(as.factor(bird_se$code_sp))[1:5]]),
                 .(code_sp), .fun=get_ts, .progress="text")

ts_bird_se_allcountry <- ddply(bird_se_1998, .(code_sp), .fun=get_ts, .progress="text")

ts_bird_se_byreg <- ddply(droplevels(bird_se_1998[!is.na(bird_se_1998$ln_kod),]), .(code_sp, ln_kod), .fun=get_ts, .progress="text")
ts_bird_se_byreg_clean <- ts_bird_se_byreg[which(!is.na(ts_bird_se_byreg$relative_abundance)),]

ts_bird_se_byecoreg <- ddply(bird_se_1998, .(code_sp, ecoreg), .fun=get_ts, .progress="text")
ts_bird_se_byecoreg_clean <- ts_bird_se_byecoreg[which(!is.na(ts_bird_se_byecoreg$relative_abundance)),]

ts_bird_se_bysubecoreg <- ddply(bird_se_1998, .(code_sp, subecoreg), .fun=get_ts, .progress="text")
ts_bird_se_bysubecoreg_clean <- ts_bird_se_bysubecoreg[which(!is.na(ts_bird_se_bysubecoreg$relative_abundance)),]


# save outputs
saveRDS(ts_bird_se_allcountry, file = "output/ts_bird_se_allcountry.rds")
saveRDS(ts_bird_se_byreg, file = "output/ts_bird_se_byreg.rds")
saveRDS(ts_bird_se_byecoreg, file = "output/ts_bird_se_byecoreg.rds")
saveRDS(ts_bird_se_bysubecoreg, file = "output/ts_bird_se_bysubecoreg.rds")
saveRDS(glm_res, file = "output/glm_res.rds")

# plot
for(i in 1:length(levels(as.factor(bird_se$code_sp)))){
  sp <- levels(as.factor(ts_bird_se_allcountry$code_sp))[i]
  gp <- ggplot(ts_bird_se_allcountry[ts_bird_se_allcountry$code_sp==sp,],
         aes(year, relative_abundance)) + geom_line() + geom_text(x=2010, y=1, label=sp) +
    geom_line(aes(y=CI_inf), linetype="dashed") +
    geom_line(aes(y=CI_sup), linetype="dashed") +
    ylab("Relative abundance") + xlab("Years") +
    theme_modern()
  print(gp)
}
```



