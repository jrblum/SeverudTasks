## Scope
- heatcheck just writes a LRFD demand. We will need the base load cases to get both LRFD and asd

## Scope Gates
- what are the "configured limits"
- not every load case needs to be entered. For instance, we often dont look at seismic very early on since we are on the east coast. This requirement would force us to look at that and expend effort that we dont realistically need to.
	- it should just detect the load types we currently have and make the load combos that include them

## Conventions
- moments in kip-ft


## Project settings
- default to qa being net, but keep the setting
- we shouldnt need soil unit weight, overburden depth, or surcharge parameters. This is purely a bearing check at the soil interface. settlement wouldnt be in the scope of this.
- there really shouldnt be a min or max size to the footings. the size increment of 6"  and the min thickness of 12" can stay though.
- user should be able to just pick from any reinforcing size. This should not be a setting.

## Write back to revit
- for now just the footing size. We do not have custom parameters set up for the reinforcing, design status, design util, or design hash.
- i need the import rules in this model described in more plain text to me than they are described now
- we want to resize the footing geometry from the tool, it is the whole point of this

## Design tool
- the idea behind the design tool is this. We import the loads, column, bearing info, code version, and current footing info that we have (eccentricity, f'c, etc) into the web app from revit. We then list all of those in the web app in a table sort of like an excel table. The last column will be a drop down that lets you chose any of the "existing standard footings" that correspond to that code version, f'c, and qa. If there are none, or if the user just wants to, they can use a separate part of the tool that can create a custom footing and look at it. This custom footing designer will only use bottom each way reinforcing (for simplicity). The idea is that the standard footings already have a max bearing capacity that they are good for, so there isnt any calculations that we need to perform in the back end. The footings have already been verified in house for it. The drop down will include any custom footings the user has made. There will be a DCR column in the table that updates whenever a new column is selected in the drop down. It is really important that the tool never chooses a design for the engineer. The engineer must chose the right size they are looking for, so that they are still the designer.

