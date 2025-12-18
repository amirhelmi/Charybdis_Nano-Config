# Add the drivers directory to the build if the PMW3610 is enabled
if(CONFIG_PMW3610)
  add_subdirectory(drivers/sensor/pmw3610)
endif()
