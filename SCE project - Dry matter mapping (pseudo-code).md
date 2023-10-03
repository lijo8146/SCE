# 1. Download images using Sentinel-2 API
- Check the codes in https://github.com/earthlab/bridges_to_prosperity_ML
  #imports

import os
import glob
from sentinelsat import SentinelAPI, read_geojson, geojson_to_wkt
from datetime import date
from osgeo import gdal
from osgeo import gdalconst
import numpy as np
import rasterio
import matplotlib.pyplot as plt
import json

# download Sentinel-2 data by date and bounding box
# lat_max = top_value
# lat_min = bottom_value
# lon_max = right_value
# lon_min = left_value


def download_sentinel_data(username, password, start_date, end_date, bounding_box, output_directory):
    """
    Downloads Sentinel-2 data using the Copernicus API Hub.

    Args:
        username (str): Your Copernicus Open Access Hub username.
        password (str): Your Copernicus Open Access Hub password.
        start_date (str): Start date in 'YYYYMMDD' format (e.g., '20220101').
        end_date (str): End date in 'YYYYMMDD' format (e.g., '20220131').
        bounding_box (list): Bounding box coordinates in the format [lon_min, lat_min, lon_max, lat_max].
        output_directory (str): Directory where downloaded data will be stored.

    Returns:
        None
    """
    # Initialize the Sentinel API with your credentials
    api = SentinelAPI(username, password, 'https://apihub.copernicus.eu/dhus')

    # Convert the bounding box to Well-Known Text (WKT) format
    footprint = geojson_to_wkt({
        "type": "Polygon",
        "coordinates": [bounding_box]
    })

    # Search for Sentinel-2 products within the specified date range and bounding box
    products = api.query(footprint,
                         date=(start_date, end_date),
                         platformname='Sentinel-2',
                         cloudcoverpercentage=(0, 30))  # You can adjust the cloud cover percentage as needed

    # Download the products
    for product_id, product_info in products.items():
        print(f"Downloading product {product_id}...")
        api.download(product_id, directory_path=output_directory)

if __name__ == "__main__":
    # Set your Copernicus Open Access Hub credentials
    username = "enter_username"
    password = "enter_password"

    # Define the date range and bounding box for your area of interest
    start_date = "20220101"
    end_date = "20220131"
    bounding_box = [-120.529210, 33.183798, -114.131493, 38.405490]  # bounding box coordinates projected in NAD 1983
    output_directory = r"C:\Users\lilly\Documents\Sentinel"  # desired output directory change to AWS

# 2. Resample bands to 10-m resolution
- Select bands 2 to 8A and 11 to 12 (10 bands total)
- Resample the 20-m resolution bands to 10-m resolution 
- Stack the 10 bands into one raster
- Gdal (https://gdal.org/api/python_bindings.html) does everything listed above
# downscale layers to 10m resolution


def convert_sentinel_bands_to_10m_resolution(input_directory, output_directory):
    """
    Converts Sentinel-2 bands to 10-meter resolution.

    Args:
        input_directory (str): Directory containing Sentinel-2 data.
        output_directory (str): Directory where converted data will be stored.

    Returns:
        None
    """
    # List all Sentinel-2 band files in the input directory
    band_files = glob.glob(os.path.join(input_directory, 'B*.jp2'))

    # Create the output directory if it doesn't exist
    os.makedirs(output_directory, exist_ok=True)

    for band_file in band_files:
        # Define the output file path
        output_file = os.path.join(output_directory, os.path.basename(band_file))

        # Open the input band file
        ds = gdal.Open(band_file, gdalconst.GA_ReadOnly)

        # Perform resampling and reprojection to 10-meter resolution
        gdal.Warp(output_file, ds, xRes=10, yRes=10, resampleAlg=gdalconst.GRA_Bilinear)

        # Close the input dataset
        ds = None

if __name__ == "__main__":
    # Define input and output directories
    input_directory = "path_to_input_directory"  # Replace with the directory containing Sentinel-2 data
    output_directory = "path_to_output_directory"  # Replace with your desired output directory

    # Call the convert_sentinel_bands_to_10m_resolution function
    convert_sentinel_bands_to_10m_resolution(input_directory, output_directory)

# mask out clouds and water using the SCL attribute of Sentinel data
# mask out clouds and water using the SCL (Scene Classification Layer) attribute of Sentinel-2 data, you can create a function that reads the SCL band, applies a mask to identify cloudy and water areas, and then applies this mask to the other bands. 


def mask_clouds_and_water(input_directory, output_directory):
    """
    Masks out clouds and water in Sentinel-2 data using the SCL band.

    Args:
        input_directory (str): Directory containing Sentinel-2 data.
        output_directory (str): Directory where masked data will be stored.

    Returns:
        None
    """
    # List all Sentinel-2 band files in the input directory
    band_files = glob.glob(os.path.join(input_directory, 'B*.jp2'))

    # Create the output directory if it doesn't exist
    os.makedirs(output_directory, exist_ok=True)

    # Determine the SCL file path (assuming it's in the same directory)
    scl_file = os.path.join(input_directory, 'SCL.jp2')

    for band_file in band_files:
        # Define the output file path
        output_file = os.path.join(output_directory, os.path.basename(band_file))

        # Open the input band and SCL files
        ds_band = gdal.Open(band_file, gdalconst.GA_ReadOnly)
        ds_scl = gdal.Open(scl_file, gdalconst.GA_ReadOnly)

        # Read band data as a numpy array
        band_data = ds_band.ReadAsArray()

        # Read SCL data as a numpy array
        scl_data = ds_scl.ReadAsArray()

        # Create a mask for cloudy and water pixels
        # In the SCL band, value 3 corresponds to cloud and value 6 corresponds to water
        cloud_water_mask = np.logical_or(scl_data == 3, scl_data == 6)

        # Apply the mask to the band data by setting masked values to a nodata value (e.g., 0)
        band_data[cloud_water_mask] = 0

        # Create an output GeoTIFF file with the same metadata as the input band
        driver = gdal.GetDriverByName("GTiff")
        output_ds = driver.Create(output_file, ds_band.RasterXSize, ds_band.RasterYSize, 1, gdalconst.GDT_Float32)
        output_ds.SetGeoTransform(ds_band.GetGeoTransform())
        output_ds.SetProjection(ds_band.GetProjection())
        output_ds.GetRasterBand(1).WriteArray(band_data)

        # Close the datasets
        ds_band = None
        ds_scl = None
        output_ds = None

if __name__ == "__main__":
    # Define input and output directories
    input_directory = "path_to_input_directory"  # Replace with the directory containing Sentinel-2 data
    output_directory = "path_to_output_directory"  # Replace with your desired output directory

    # Call the mask_clouds_and_water function
    mask_clouds_and_water(input_directory, output_directory)

# 3. Create spectral libraries
- Collect spectra across seasons and years (> 50 per class)
- Classes: each vegetation type's green vegetation and dry vegetation, bare soil, water and shade
- Collect spectra at the pixel level (capture the variability within classes, sunlight pixels preferentially - we will have the shade class)
- NEON R code to be translated to Python: https://github.com/earthlab/neonhs
- May use Python functions from this: https://www.spectralpython.net/

# 4. Create metadata
- A final, merged spectral library must have a metadata (.csv) with at least spectrum ID (column 1), class (2), and sub-class (3) description. 
- For the vegetation type classes the order is: 1. class: green vegetation or non-photosynthetic vegetation (i.e. dry matter); 2. sub-class: vegetation type. For the other classes (bare soil, water..), repeat the class content in the sub-class column.

# 5. Endmember Selection
- TBD

# 6. Multiple Endmber Spectral Mixture Analysis Emulation
- TBD

# 7. Dry matter Index Calculation
