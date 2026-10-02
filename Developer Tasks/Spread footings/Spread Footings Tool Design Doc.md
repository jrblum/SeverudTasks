
### Inputs
 - Footing f'c
 - Column f'c
 - Column Reaction
 - Footing Load
 - Column eccentricity on footings (ex & ey)
 - Max allowable bearing pressure (qa)
 - column depth and width (h & b)
 - foundation and column locations
#### Where do the inputs come from?
- revit
- user input (qa)
- user override?
- calculation using other inputs
	- ex and ey calculated using the foundation and column locations
		- or is this a user input?

### Data Transfer
- Revit app writen in C#
- outputs a .csv with all the footing / column information we need
	- is it two csvs? one with col and one with foundations?
	- what about a json?
- choose the location where the csv is exported (similar to my personal ram scripts where a gui pops up to get the .rss path)
- transfer the data back to revit using those csvs?
	- how do we make sure we are updating correct columns and footings?
		- guid?

### APP
#### Brains
- can do both one-offs and bulk design
- the one-off is the main calc engine.
	- this engine can do concentric footings, eccentric within the kern footings
		- down the line add functionality for footings with tie beams
- add in feature to show loading / internal force diagrams
	- internal moment and shear can be found a few ways
		1. singularity functions
		2. opensees
		3. rhino + karamba?
- has custom footing functionality, but also can just use the severud standard spread footings that benji is working on
#### Architecture
- web app in python
	- necessary dependancies
		- streamlit (for gui)
		- numpy
		- pandas
	- future dependencies
		- shapely(?) for displaying loadings
		- openseespy?
- IMPORTANT:
	- actual column design must not be in REVIT. keep the philosophy that revit is just for modeling, not for design

