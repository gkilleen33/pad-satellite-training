# Satellite yield measurement - PAD training

This repository contains training materials for the construction of satellite yield measurements using multispectral satellite imagery from the Sentinel-2 mission.

The materials were constructed for an internal training in Precision Agriculture for Development. The focus of the exercise is on acquiring and processing Sentinel-2 imagery, calculating vegetation indices from satellite imagery locally using Python and on the Google Earth Engine, masking images, and calculating zonal statistics.

The exercise uses crop cut data from maize and sorghum plots in Uganda acquired via the Radiant MLHub.

This repository is only intended for training, and the results should not be taken as accurate crop yield measurements. There are imperfections in several areas of the method, including cloud masking and the way that yield measurements are constructed from the time series of VI values for each plot. The imagery used may also not be in the ideal time period for measuring the productivity of these crops. These decisions were primarily made in the interest of clarity and brevity in the training and not necessarily to optimize the quality of yield predictions.

The satellite VIs calculated are ultimately not very predictive of the crop cut data. This may reflect issues in the remote sensing methodology, flaws in the crop cut data, flaws in the plot boundary data, or a combination of the factors.

If you observe any flaws in the tutorial, please contact Grady Killeen by submitting an issue or via email.

# Instructions

This tutorial begins by downloading and processing the plot boundary, crop cut, and satellite data in the notebook `01-get-process-data.ipynb`. Second, vegetation indices are calculated for each plot and basic analysis is performed in `02-analysis.ipynb`. Finally, this work is re-created using the Google Earth Engine in `03-google-earth-engine.ipynb`.

The tutorial requires the [Anaconda](https://www.anaconda.com/) package manager for Python. If you attempt to use a different package manager, it almost certainly will not work because a lot of geospatial Python packages struggle to build outside of Anaconda.

My `.condarc` file is configured as follows:
```
channels:
  - defaults
  - conda-forge
channel_priority: flexible

ssl_verify: true
```

This allows packages to install from both the default channels and conda-forge. If you wish to manually create your own environment, consider changing the configuration to match this, and always try to install GIS packages from conda-forge before default channels. GDAL in particular must be installed from conda-forge for this to work.

Alternatively, you may try navigating to the directory containing the `environment.yml` file in this repository and then entering `conda env create --file environment.yml`. This should create a fully functioning environment that can be activated by entering `conda activate gpd`. However, this may not work over time as specific package versions are removed.

The tutorial also requires the program [Sen2Cor](https://step.esa.int/main/third-party-plugins-2/sen2cor/) which creates surface reflectance (L2A) Sentinel-2 images from L1C data and generates a more accurate scene classification for filtering out clouds. I used version 2.8. The program should be added to your path so that you can call it by typing `L2A_Process --help` from any command line. My L2A_GIPP.xml was configured as follows:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Level-2A_Ground_Image_Processing_Parameter xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="L2A_GIPP.xsd">
  <Common_Section>
    <Log_Level>INFO</Log_Level>
    <!-- can be: NOTSET, DEBUG, INFO, WARNING, ERROR, CRITICAL -->
    <Nr_Threads>AUTO</Nr_Threads>
    <!-- Nr_Treads determines the number of threads used for reading the OpenJPEG2 images. This is a new
         feature implemented with OpenJPEG 2.3., improving the speed for importing the Bands.
         If AUTO is chosen, the number of treads are deduced, using cpu_count().
         Set this to 1 up to a maximum of 8, if this automatic mode will not fit to your platform -->
    <DEM_Directory>SRTM</DEM_Directory>
    <!-- should be either a directory in the sen2cor home folder or 'NONE'. If NONE, no DEM will be used -->
    <DEM_Reference>http://data_public:GDdci@data.cgiar-csi.org/srtm/tiles/GeoTIFF/</DEM_Reference>
    <!-- disable / enable the upper two rows if you want to use an SRTM DEM -->
    <!-- The SRTM DEM will then be downloaded from this reference, if no local DEM is available -->
    <!-- if you use Planet DEM you can optionally add the local path instead,
         which then will be inserted in the datastrip metadata -->
    <Generate_DEM_Output>FALSE</Generate_DEM_Output>
    <!-- FALSE: no DEM output, TRUE: store DEM in the AUX data directory -->
    <Generate_TCI_Output>TRUE</Generate_TCI_Output>
    <!-- FALSE: no TCI output, TRUE: store TCI in the IMAGE data directory -->
    <Generate_DDV_Output>FALSE</Generate_DDV_Output>
    <!-- FALSE: no DDV output, TRUE: store DDV in the QI_DATA directory -->
    <Downsample_20_to_60>TRUE</Downsample_20_to_60>
    <!-- TRUE: create additional 60m bands when 20m is processed -->
    <PSD_Scheme PSD_Version="14.2" PSD_Reference="S2-PDGS-TAS-DI-PSD-V14.2_Schema">
		<UP_Scheme_1C>S2_User_Product_Level-1C_Metadata</UP_Scheme_1C>
		<UP_Scheme_2A>S2_User_Product_Level-2A_Metadata</UP_Scheme_2A>
		<Tile_Scheme_1C>S2_PDI_Level-1C_Tile_Metadata</Tile_Scheme_1C>
		<Tile_Scheme_2A>S2_PDI_Level-2A_Tile_Metadata</Tile_Scheme_2A>
		<DS_Scheme_1C>S2_PDI_Level-1C_Datastrip_Metadata</DS_Scheme_1C>
		<DS_Scheme_2A>S2_PDI_Level-2A_Datastrip_Metadata</DS_Scheme_2A>
    </PSD_Scheme>
    <PSD_Scheme PSD_Version="14.5" PSD_Reference="S2-PDGS-TAS-DI-PSD-V14.5_Schema">
		<UP_Scheme_1C>S2_User_Product_Level-1C_Metadata</UP_Scheme_1C>
		<UP_Scheme_2A>S2_User_Product_Level-2A_Metadata</UP_Scheme_2A>
		<Tile_Scheme_1C>S2_PDI_Level-1C_Tile_Metadata</Tile_Scheme_1C>
		<Tile_Scheme_2A>S2_PDI_Level-2A_Tile_Metadata</Tile_Scheme_2A>
		<DS_Scheme_1C>S2_PDI_Level-1C_Datastrip_Metadata</DS_Scheme_1C>
		<DS_Scheme_2A>S2_PDI_Level-2A_Datastrip_Metadata</DS_Scheme_2A>
	</PSD_Scheme>
    <GIPP_Scheme>L2A_GIPP</GIPP_Scheme>
    <SC_Scheme>L2A_CAL_SC_GIPP</SC_Scheme>
    <AC_Scheme>L2A_CAL_AC_GIPP</AC_Scheme>
    <PB_Scheme>L2A_PB_GIPP</PB_Scheme>
  </Common_Section>
  <Scene_Classification>
    <Filters>
      <Median_Filter>0</Median_Filter>
    </Filters>
  </Scene_Classification>
  <Atmospheric_Correction>
    <Look_Up_Tables>
      <Aerosol_Type>RURAL</Aerosol_Type>
      <!-- RURAL, MARITIME, AUTO -->
      <Mid_Latitude>SUMMER</Mid_Latitude>
      <!-- SUMMER, WINTER, AUTO -->
      <Ozone_Content>331</Ozone_Content>
      <!-- The atmospheric temperature profile and ozone content in Dobson Unit (DU)
      	0: to get the best approximation from metadata
      	(this is the smallest difference between metadata and column DU),
      	else select one of:
      	==========================================
        For midlatitude summer (MS) atmosphere:
        250, 290, 331 (standard MS), 370, 410, 450
        ==========================================
        For midlatitude winter (MW) atmosphere:
        250, 290, 330, 377 (standard MW), 420, 460
        ==========================================
       -->
    </Look_Up_Tables>
    <Flags>
      <WV_Correction>1</WV_Correction>
      <!-- 0: No WV correction, 1: only 940 nm bands, 2: only 1130 nm bands , 3: both regions used during wv retrieval, 4: Thermal region -->
      <VIS_Update_Mode>1</VIS_Update_Mode>
      <!-- 0: constant, 1: variable visibility -->
      <WV_Watermask>1</WV_Watermask>
      <!-- 0: not replaced, 1: land-average, 2: line-average -->
      <Cirrus_Correction>FALSE</Cirrus_Correction>
      <!-- FALSE: no cirrus correction applied, TRUE: cirrus correction applied -->
      <DEM_Terrain_Correction>TRUE</DEM_Terrain_Correction>
      <!--Use DEM for Terrain Correction, otherwise only used for WVP and AOT -->
      <BRDF_Correction>0</BRDF_Correction>
      <!-- 0: no BRDF correction, 1, 2, 11, 12, 22, 21: see IODD for explanation -->
      <BRDF_Lower_Bound>0.22</BRDF_Lower_Bound>
      <!-- In most cases, g=0.2 to 0.25 is adequate, in extreme cases of overcorrection g=0.1 should be applied -->
    </Flags>
    <Calibration>
      <Adj_Km>1.000</Adj_Km>
      <!-- Adjancency Range [km] -->
      <Visibility>40.0</Visibility>
      <!-- visibility (5 <= visib <= 120 km) -->
      <Altitude>0.100</Altitude>
      <!-- [km] -->
      <Smooth_WV_Map>100.0</Smooth_WV_Map>
      <!-- length of square box, [meters] -->
      <WV_Threshold_Cirrus>0.25</WV_Threshold_Cirrus>
      <!-- water vapor threshold to switch off cirrus algorithm [cm]Range: 0.1-1.0 -->
      <Database_Compression_Level>0</Database_Compression_Level>
      <!-- zlib compression level for image database [0-9, 0: best speed, 9: best size] -->
    </Calibration>
  </Atmospheric_Correction>
</Level-2A_Ground_Image_Processing_Parameter>
```

I also suggest installing QGIS or ArcGIS which are not used directly, but are useful for examining imagery and outputs. The [Sentinels Application Platform (SNAP)](https://step.esa.int/main/toolboxes/snap/) is also useful to verify the integrity of images and calculate vegetation indices to make sure that the Python code does not have errors.

You will need to obtain credentials to the [Copernicus Open Access Hub](https://scihub.copernicus.eu/) to acquire the Sentinel-2 imagery, the [Radiant MLHub](https://www.mlhub.earth/) to download the crop cut/plot boundary data, and register for the [Google Earth Engine](https://earthengine.google.com/).

Finally, you must install the Google Earth Engine Python API through Anaconda for local use. Instructions are available [here](https://developers.google.com/earth-engine/python_install-conda).

Please note that it took me over a week to download the Sentinel-2 imagery because images older than a year old were moved to the Long Term Archive. You may wish to find the imagery through a different service, such as a mirror.

# Folder structure

You should create the following folder structure, relative to the cloned repository, before beginning the tutorial. I did not always include make directory commands, so it otherwise may not run.

```
.
├── imagery
│   ├── composites
│   │   └── fc-tiles
│   ├── final
│   ├── l1c
│   ├── l2a
│   └── vis
├── uganda-crops
│   ├── merged
│   └── raw
└── zonal-stats
```

If you are able to acquire the Sentinel-2 data from another source, you should insert each product as a zip file in the `imagery/l1c` directory.
