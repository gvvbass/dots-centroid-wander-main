# Version Information

## Current Version: v2.0 (Recommended)

PASTAM is currently at version 2.0, which includes significant improvements over the original implementation.

## Version History

### v2.0 (Current)
- **Location**: `src/v2.0/`
- **Status**: Active development, recommended for all new projects
- **Key Features**: Progress reporting, quality analysis, bug fixes
- **Documentation**: `src/v2.0/README_v2.md`

### v1.0 (Legacy)
- **Location**: `src/v1.0/`
- **Status**: Preserved for backward compatibility
- **Usage**: Existing projects using global variables
- **Documentation**: `src/v1.0/README_v1.md`

## Which Version Should I Use?

**For new projects**: Use v2.0
- Better performance
- More features
- Active support
- Modern interface

**For existing v1.0 projects**: 
- Continue with v1.0 if it works
- Migrate to v2.0 when convenient (see `docs/migration_guide.md`)
- Migration is straightforward (mostly search-replace)

## Quick Start by Version

### v2.0 (Recommended)
```matlab
addpath(genpath('src/v2.0'));
TM = centroidwander(A_obj, 3, 3, 'median', 0.01);
stats = analyzeTrackingQuality(TM);
```

### v1.0 (Legacy)
```matlab
addpath('src/v1.0');
global TM;
centroidwander(A_obj, 3, 3, 'median', 0.01, 'TM');
```

See `CHANGELOG.md` for complete list of changes.
