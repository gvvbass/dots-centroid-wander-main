# Changelog

All notable changes to PASTAM will be documented in this file.

## [v2.0.0] - 2025-11-18

### Added
- Progress reporting with ETA estimation
- Structured anomaly logging (table format)
- `analyzeTrackingQuality()` function for comprehensive analysis
- Quality score metric (0-100)
- Spot count validation
- Optional debug visualization
- Parameterized configuration via name-value pairs
- Final state validation checks
- Iterative convergence for anomaly recovery
- Comprehensive documentation

### Changed
- **BREAKING**: Function now returns TM instead of using global variable
- **BREAKING**: Anomaly structure changed from cell array to table
- Improved coordinate system handling for edge detection
- Enhanced temporal validation with explicit logging
- Unified threshold management across matching and recovery

### Fixed
- Frame counting bug (was using `length()` instead of `size(,3)`)
- Global variable dependency removed
- Zero-dimension box handling
- Coordinate convention inconsistencies
- Recovery threshold decoupling from main threshold

### Deprecated
- v1.0 global variable interface (still available in `src/v1.0/`)

## [v1.0.0] - Original Release

### Features
- Adaptive thresholding spot detection
- Frame-to-frame bounding box matching
- Temporal validation (false positive removal)
- Anomaly recovery (fusion, split, missing spots)
- Support for FITS and video files
- PASTAM class structure

### Known Issues
- Uses global variables
- Frame counting bug with non-square arrays
- Inconsistent anomaly logging
- No progress reporting
