Global-to-Regional Integrated forecast SysTem (GRIST)
================================================================================
GRIST is a global model designed for flexible and multi-purpose usages in weather and climate modeling research and applications. The model is developed via rigorous scientific (experimental protocols/metrics) and technical (regresssion/self-consistency) tests in a hierarchical manner, covering spectra of experimental protocols for global atmospheric modeling. It has been used for experimental earth weather and climate modeling applications.  

<p align="left">
  <img src = "doc/picture/2020-01-22-utc3.png">
  Global Storm-Resolving Simulations by GRIST Nonhydrostatic Model during DYAMOND winter.  
  Visualization by Florian Ziemen/ESiWACE2/DKRZ
</p>

Table of Contents 
--------------------------------------------------------------------------------
- [User_guide](#user_guide)
- [Code Layout](#code)
- [Build scripts](#build)
- [Running Mode](#run)
- [Input Data](#input_data)
- [Output Data](#output_data)
- [Post process](#post_process)
- [Known issues](#known_issues)
- [Configuations](#configurations)
- [Developers](#Developers)
- [Acknowledgment](#acknowledgment)
- [Reference](#reference)

User_guide
-------------------------------------------------------------------------------- 
+ See BUILD.txt for preparing external software libraries; how to build the model code for a target executable file.  
+ Prepare necessary input data (cf., inputdata).  
+ Follow run/xxx with some tested configurations to run the model. Check post_process for evaluating data. Recommend to first run HDC-G6 for one year. Use provided data to ensure your model porting procedure is correct. (Some reference output data will be added soon).       
+ Minimal requirements one would require to run a full-physics coupled GRIST from scratch (e.g., AMIPW):  
(i) initial data (atm and land);  
(ii) static data;  
(iii) grid file;  
(iv) sst and sic data;  
(v) data related to the physics suite;  
(vi) model code (ParGRIST), four external libraries (Lapack, METIS, pnetcdf, netcdf), compilers;  
(vii) namelist files;  

Code
--------------------------------------------------------------------------------
+ Please always download a tagged/released version.  
+ Master branch: the major branch for model development. Other branches are used for dev, testing and ref.  
+ See grist-structure.png in doc for an overview of code structure.  
+ Due to the fast pace of model development, certain (typically only few) functions of the model may not be simply used out-of-box, but the necessary changes are mostly trivial.   

Build
--------------------------------------------------------------------------------
+ build_amipw: build GRIST with WRF-derived physics (AMIP with a 'W'eather-style physics suite)
+ build_amipc: build GRIST with CAM-derived physics (AMIP with a 'C'limate-style physics suite)
+ build_gcm  : build GRIST with simple-physics at most (the most simple 3D atmosphere model)
+ build_scm  : build SCM model of GRIST
+ build_swe  : build SWM model of GRIST
+ The code can be seperately compiled for model framework+dynamics (until dtp mode); and full-physics (amp mode). Compiling and running framework+dynamics should not depend on any specific physpkg except simple-physics.  

When building the four external libs (netcdf, pnetcdf, metis, Lapack), in parcicular, Lapack, please make sure you have passed all its self-check tests and zero error is produced. Please also make sure they are compiled using the same compiler as building the model.  

Run
--------------------------------------------------------------------------------
+ working_mode=amipw: run GRIST 3D atmosphere with WRF derived physics.  
+ working_mode=amipc: run GRIST 3D atmosphere with CAM derived physics.  
+ working_mode=dycore, tracer, dtp: run GRIST 3D atmosphere model in idealized test cases.  
+ model=scm: use SCM model  
+ model=swe: use SWE model    
+ By default, building GRIST with full-physics implies test_real_case=.true. or doAquaPlanet=.true. Other working_modes and simple models (scm, swe) require code rebuild with case-specific compiling options.   

Input_Data
--------------------------------------------------------------------------------
##### General Info
Sample inputdata for various grid resolutions and common physics/lsm data are ongoing available at: https://pan.baidu.com/s/1oQnOMDltTKADiaVnqMHbcg (code: ffqs). Both mesh and static files only need to be generated once for each resolution. Everything will change if the mesh location changes. Frequently-used mesh and static files are ongoing provided at that link.

##### Mesh file & Surface static file
Please use a pre-generated mesh file at  https://pan.baidu.com/s/1oQnOMDltTKADiaVnqMHbcg. An incomplete list would be:  
Uniform-mesh: G6, G7, G8, G9, G10, G9B3, ... ;  
VR-mesh: G5B3X4, X20EA, G8X16L4EA, G7X16L4EA, G6B3X16L4EA,...

##### SST(SIC) file
Unit: SST, Kelvin, SIC: fractional (not percent).  

The default SST/SIC file comes from input4mip preprocessed data (https://zenodo.org/record/5025234). These mid-point monthly sst data are internally linearly interpolated in time to produce daily values used by the model. The interpolation procedure (and the way how the mid-point data were modified) guarrantees that the monthly-mean of the interpolated daily values RECOVER the raw UNprocessed SST data provided by input4mip (VERIFIED).  

Three stream modes are supported, controlled by "real_sst_style", "numMonSST" nml option.   

a. cycle sst: Use a 12-month SST file. The time dimention is cycled in the model. This is mainly for climatological SST run, but can also be used for simple tests when varying-sst is not a must. E.g., if 12-month SST data are the same field, this cheap linear interpolation will give the same result, so no code change is needed for short-term NWP forecast. For this mode, SST/SIC data must have 12 time slices.  
When using invariant SST/SIC for NWP runs, one may generate SST/SIC from the initial file, see here: https://github.com/GRIST-Dev/GRIST_Data/tree/master/init/geniniFromGFS/data.   
b. AMIP sst: Each file contains an integer decade plus two neighboring months (e.g., 196912,1970-1979,198001). The model will read the target month based on the naming conventions of the SST file. For this mode, SST/SIC data must have 1+120+1=122 time slices. If data are not enough, use fake data to complete the time.  

c. DAILY sst: Eac file contains a daily sst value for a time slice. read at the utc00 of this day. (This can be generalized to read at every-N-step in future)

Note: if use_som = .true., SST (but not SIC) will be replaced by that from a slab ocean model at each physics step. This may be a choice for medium-range to seasonal real-time forecast.  

+ See here for generating SST to a target mesh: https://github.com/GRIST-Dev/GRIST_Data/tree/master/genSST.   
+ If the sst/sic data has missing values over land, set sst's missing value to 271.38, sic to 0 before remapping.  
+ It has been checked that using CDO's remapycon and NCL's bilinear interpolation function produce very-close interpolated results. CDO is recommended for its speed. This requires to write a CDO remaping template file (SCRIP-style is recommended; see scripts/SCRIP_cdo/) based on the mesh file. Meanwhile, for real-time forecast, it is better to first modify the raw SST/SIC, then remap it to the target model mesh.  

##### Initial file for ATM and LND
(i) CDO-based approach. See here for an example using ERA5 renalysis datasets:   
https://github.com/GRIST-Dev/GRIST_Data/tree/master/init/geniniFromERA5.  
While this workflow is demonstrated using ERA5, the entire approach can be generalized for any Analysis/Re-analysis data from operational centers. When using land initial data from ERA5, set grist_lsm_noahmp.nml as:    
&lsm_soil_layers_nl   
  num_soil_layers                   = 4  
  soil_thick_input(1)               = 0.035  
  soil_thick_input(2)               = 0.175  
  soil_thick_input(3)               = 0.64   
  soil_thick_input(4)               = 1.775   
/  
A GFS-based CDO workflow (https://github.com/GRIST-Dev/GRIST_Data/tree/master/init) is also added for GFS-based raw data (https://www.ncdc.noaa.gov/data-access/model-data/model-datasets/global-forcast-system-gfs):  
Use the following soil layers when using its LAND data:  
&lsm_soil_layers_nl   
  soil_thick_input(1)               = 0.05  (0.1)   
  soil_thick_input(2)               = 0.25  (0.3)  
  soil_thick_input(3)               = 0.70  
  soil_thick_input(4)               = 1.50  
/   

Note: ERA5 and GFS have different land initial states for the same date (certain variables like SNOW and SNOWH are largely different; others are moderately different). But short-term NWP tests using these two LND initial files (with a common ATM initial file) indicate that they produce very close solutions. See comp-gfs-era-land.crop.png. A 2x2 comparison (ERA,GFS for ATM&LND) is ERA5-GFS-G6-compare.png in doc.  

##### Offline partition data (optional)   
For some very high-resolution runs (e.g., G9B3), using an offline partition strategy will largely reduce the initilization time. This produces same results as one use an online partition (by default). partition.exe will write some binary partition information before running the model. In this case, one need to first partition the global mesh to a FIXED number of decomposition. If this is needed, use partition.exe and related grist.nml to generate the partition files. Then set read_parition = .true. and pardir='${where your partition files are located}' in the grist.nml.  

The partition files are reusable only in the sense that: the number of decomposed domains does not change; the binary files are generated using the same compiling options as one builds the model exe.  

Output_Data
--------------------------------------------------------------------------------
Sample output files look like when ${working_mode}=amipw,amipc  

1. Monthly mean (h0) diagnostic output:  
GRIST.ATM.G6.${working_mode}.MonAvg.2000-05.1d.h0.nc  
GRIST.ATM.G6.${working_mode}.MonAvg.2000-05.2d.h0.nc  
2. N-step (h1) diagnostic output:  
GRIST.ATM.G6.${working_mode}.2007-07-07-00000.1d.h1.nc  
GRIST.ATM.G6.${working_mode}.2007-07-07-00000.2d.h1.nc  
GRIST.ATM.G6.${working_mode}.2007-07-07-00000.3d.h1.nc  
GRIST.LND.G6.${working_mode}.2007-07-07-00000.1d.h1.nc     
3. Restart output (only support amipw now):  
GRIST.RST.Dyn.G6.1d.amipw.1980-05-01.00850-01200.nc   
GRIST.RST.Dyn.G6.2d.amipw.1980-05-01.00850-01200.nc  
GRIST.RST.Dyn.G6.3d.amipw.1980-05-01.00850-01200.nc  
GRIST.RST.Phy.G6.1d.amipw.1980-05-01.00850-01200.nc   
GRIST.RST.Phy.G6.2d.amipw.1980-05-01.00850-01200.nc  
GRIST.RST.Lnd.G6.1d.amipw.1980-05-01.00850-01200.nc  
GRIST.RST.Lnd.G6.2d.amipw.1980-05-01.00850-01200.nc  

+ Here 00850 indicates ${whole_integrated_days}, 01200 indicates model_timestep. Set 86400/${model_timestep}x${whole_integrated_days}+1 in step_restart.txt for restarting the model at the target date.  
+ The h0&h1 diagnostics can be used together; but if one does not want to be overwhelmed by uncessary data at certain seasons (e.g., only some months are focused), use the restart approach for intensive output.  
+ A refactored IO frequency for h1_history is added, to allow different output frequencies for 1d, 2d, 3d variables. Note that for different outpue freq, the instant var will not be affected, while avg var marked by "xxxDiag" will have different values for the same date if output freq is changed. Because the time-period for accumulating these avg-variables depends on the h1_1/2/3d_history_freq.  

Post_process
--------------------------------------------------------------------------------
see data directory for some sample post-processing workflow.

Known_issues
--------------------------------------------------------------------------------
1. For restart runs, currently the first-generated file as in a monthl-mean output should be discarded because the accumulated-mean state does not include the first restart time. The model only ensures correct instant states. From the 2nd one, everything will be fine. If restart is needed, one would be better to start at a time several slices ahead the actual target time to ensure the correctness of diagnostic output.  
2. This problem seems to be cured if upgrade to intel 2018+pnetcdf1.12.2 at PI, but still documented here for reference. [On CMA's Sugon-PI, the parallel I/O only works for certain comm_group_size number. Typically setting it to ${nprocessor} or setting to 1 will work. This is an unusual behavior not seen in some other machines. However, when it works, the parallel I/O is efficient for even global 3.5 km run (Glevel 11).]   
3. Current default AMIP sst data only go to 2016. We use some fake data to satisfy the data format requirement.  
4. The WRF physics modules produce different machine solution with or without -fopenmp (not bit-to-bit reproducible). This is not a problem of 'openmp parallization by GRIST', but an issue with certain WRF physics modules (even withoug omp activated, this problem still exists). Simple physics and CAM5 physics does not have this issue. For default use of WRF physics modules, do not use -fopenmp or -qopenmp in your compiling options.

Developers
-------------------------------------------------------------------------------- 
+ Frequently used self-check tests to ensure technical correctess.  
1. Running the code at different degrees of parallelization produces identical netCDF output (e.g. "cdo diff" does not complain).  
2. Running the code in a restart mode produces identical results as in a initial run.  
3. After a code refacoring without scientific changes, the new code must exactly reproduce the results of the raw-unmodified code.  

Configurations
--------------------------------------------------------------------------------
1. Tracer transport (tracer_para)   
The latest namelist (grist_nml_module) contains some combinations for different possible tracer transport options supported by GRIST. Not every combination is tested in real world setup.  
2. Dycore setup (dycore_para)   
Do not recommend to modify dycore setup except default configurations for each resolution. Typical changes for each-resolution in real world include horizontal diffusion (smg, hyper), time step, etc.    
3. Horizontal resolution  
The model has been tested from 120 km to 3.75 km uniform-resolution in real world setup. Some VR grid is also tested for NWP run. We suggest to first select a horizontal resolution grid then just use and tune the model at this resolution.  
4. Vertical resolution  
The current default vertical resolution for real-world is L30 with a top at 2.25 hPa (40 km). There are some testing options in grist_hpe_constants.F90 (e.g., L60). But not every one is verified in real world setup.  
5. Physics-dynamics coupling options  
For real-world simulations, the following pdc combinations are currently supported:  
HDC-AMIPW/NDC-AMIPW/HDC-AMIPC  

6. Physics suites  
6.1 AMIPW physics (Weather-style)  
Most parameters are internnaly defined in the code. Options are choosed in grist_amipw_phys.nml  
6.2 AMIPC physics (Climate-style)   
Defined in grist_amipc_phys.nml and internally in the code.  

Reference
--------------------------------------------------------------------------------
Model framework and dynamics
1. Zhang, Y., Li, J., Yu, R., Liu, Z., Zhou, Y., Li, X., and Huang, X.: A Multiscale Dynamical Model in a Dry-mass Coordinate for Weather and Climate Modeling: Moist Dynamics and its Coupling to Physics, Monthly Weather Review, 10.1175/MWR-D-19-0305.1, 2020.
2. Zhou, Y., Zhang, Y., Li, J., Yu, R., and Liu, Z.: Configuration and evaluation of a global unstructured mesh atmospheric model (GRIST-A20.9) based on the variable-resolution approach, Geosci. Model Dev., 13, 6325-6348, 10.5194/gmd-13-6325-2020, 2020.
3. Zhang, Y., Li, J., Yu, R., Zhang, S., Liu, Z., Huang, J., and Zhou, Y.: A Layer-Averaged Nonhydrostatic Dynamical Framework on an Unstructured Mesh for Global and Regional Atmospheric Modeling: Model Description, Baseline Evaluation, and Sensitivity Exploration, Journal of Advances in Modeling Earth Systems, 11, 1685-1714, 10.1029/2018MS001539, 2019.
4. Wang, L., Zhang, Y., Li, J., Liu, Z., and Zhou, Y.: Understanding the Performance of an Unstructured-Mesh Global Shallow Water Model on Kinetic Energy Spectra and Nonlinear Vorticity Dynamics, Journal of Meteorological Research, 33, 1075-1097, 10.1007/s13351-019-9004-2, 2019.
5. Zhang, Y.: Extending High-Order Flux Operators on Spherical Icosahedral Grids and Their Applications in the Framework of a Shallow Water Model, Journal of Advances in Modeling Earth Systems, 10, 145-164, 10.1002/2017MS001088, 2018.
6. Zhang, Y., Yu, R., and Li, J.: Implementation of a conservative two-step shape-preserving advection scheme on a spherical icosahedral hexagonal geodesic grid, Advances in Atmospheric Sciences, 34, 411-427, 10.1007/s00376-016-6097-8, 2017.
7. Li, J., and Y. Zhang, 2022: Enhancing the stability of a global model by using an adaptively implicit vertical moist transport scheme. Meteorology and Atmospheric Physics, 134, 55.
8. Liu, Z., Zhang, Y., Huang, X., Li, J., Wang, D., Wang, M., and Huang, X.: Development and performance optimization of a parallel computing infrastructure for an unstructured-mesh modelling framework, Geosci. Model Dev. Discuss., 2020, 1-32, 10.5194/gmd-2020-158, 2020.
9. Wang, T., L. Zhuang, J. M. Kunkel, S. Xiao, and C. Zhao, 2020: Parallelization and I/O Performance Optimization of a Global Nonhydrostatic Dynamical Core Using MPI. Computers, Materials & Continua, 63.

Model Physics and applications
1. Zhang, Y., R. Yu, J. Li, X. Li, X. Rong, X. Peng, and Y. Zhou, 2021: AMIP Simulations of a Global Model for Unified Weather-Climate Forecast: Understanding Precipitation Characteristics and Sensitivity Over East Asia. Journal of Advances in Modeling Earth Systems, 13, e2021MS002592.
2. Li, X., Zhang, Y., Peng, X., and Li, J.: Using a single column model (SGRIST1.0) for connecting model physics and dynamics in the Global-to-Regional Integrated forecast SysTem (GRIST-A20.8), Geosci. Model Dev. Discuss., 2020, 1-28, 10.5194/gmd-2020-254, 2020.
3. Li X., Yi Zhang, X. Peng, W. Chu, Y. Lin, and J. Li, Improved Climate Simulation by using a Double-Plume Convection Scheme in a Global Model, Journal of Geophysical Research-Atmospheres, 2022.
4. Zhang, Y., Z. Liu, X. Li, X. Rong, J. Li, Y. Zhou, and S. Chen, 2022: Resolution Sensitivity of GRIST Nonhydrostatic Model during DYAMOND winter from 120 km to 5 km (3.75 km).
