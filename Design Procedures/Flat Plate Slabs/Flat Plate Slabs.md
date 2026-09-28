#### Software:
-  RAM Concept (Pulled from RAM SS)


#### Steps:
1. Pull the Concept Files from RAM SS
2. Debug the mesh
3. draw the design strips
	1. General Tab Settings:
	  ![[Pasted image 20260922160252.png|393]]
	2. Strip Generation Tab Settings:
	  ![[Pasted image 20260922160709.png|394]]
	3. Column Strip Tab Settings:
	  ![[Pasted image 20260922160758.png|397]]
	4. Middle Strip Tab Settings:
	  ![[Pasted image 20260922160827.png|399]]
	5. Live Load Reduction Tab Settings:
	  ![[Pasted image 20260922160853.png|400]]
	 **Note: For coupling beams, use full width instead of code slab in the strip generation tab**
4. For both long. and lat. strips, span cross-sections to be at 90 degree angles, using the button seen below
   ![[Pasted image 20260922162311.png]]
5. For bottom steel, pin all the columns and walls
	1. For bottom steel, the column and wall settings should look like this:
	2. For top steel, the column and wall settings should look like this:
6. For top steel, fix all columns and walls
	1. For bottom steel, the column and wall settings should look like this:
	2. For top steel, the column and wall settings should look like this:
7. run the analysis with detailing
8. go to the "strip" rebar area overlay. Check that your typical steel is enough for the required reinforcing everywhere. Otherwise add extra reinforcing


#### Typical Reinforcing:
- Top Middle (TM): #4 @ 12" OC
- Top Column (T): #5 @ 12" OC
- Slab Edge Reinforcing: #4 @ 12" OC
- Perimeter Reinforcing: (2) #6
- Bottom Mat: #4 @ 12" EW
- 