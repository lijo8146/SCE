# 1. Download images using Sentinel-2 API
- Check the codes in https://github.com/earthlab/bridges_to_prosperity_ML
  
# 2. Resample bands to 10-m resolution
- Select bands 2 to 8A and 11 to 12 (10 bands total)
- Resample the 20-m resolution bands to 10-m resolution 
- Stack the 10 bands into one raster
- Gdal (https://gdal.org/api/python_bindings.html) does everything listed above

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
