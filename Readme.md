# Neruospice v1.0

## Abstract
Rapid advances in computing and neural interfacing technologies have encouraged the development of model-based approaches to predicting brain behavior. Combining meshed models with discrete solutions to the partial differential equations which govern the spread of currents forms the foundation for powerful models of electrical stimulation of neural tissue. But, as these models increase in complexity, the need becomes more severe for the development of sophisticated computational approaches to understanding the fields generated by not only active electrodes but also active neurons. 

Due to the spatial, temporal, and magnitudinal differences in electrode vs. neuron generated fields, different numerical algorithms may be required to accurately predict voltage gradients throughout affected volumes when generated by numerous neuronal sources. To address this concern, we have conducted a study which explores the impact on field estimation of mesh geometry and algorithms for handling spatial mismatches between current sources and meshed nodes. Establishment of a robust algorithm for numerical estimation of field potentials arising from neural activation could provide a path toward more predictive modeling of large-scale recording systems, neural network behavior, and extracellular neuron-neuron interactions.

Neurospice v1.0 comprises a Python library to support automated construction of a mesh and field solution with user defined current sources as the input. This library could be used to support in silico estimation of Local Field Potentials where activated neuronal models are used to generate extracellular potentials.


Please cite the following paper if you use any of the code in this repository either directly or as inspiration:

Bingham CS, Paknahad J, Girard CBC, Loizos K, Bouteiller J-MC, Song D, Lazzi G and Berger TW. (2020). Admittance Method for Estimating Local Field Potentials Generated in a Multi-Scale Neuron Model of the Hippocampus. Front. Comput. Neurosci. 14:72. doi: 10.3389/fncom.2020.00072


### Hexahedral (70µm)

![(1) Hexahderal](https://user-images.githubusercontent.com/7799699/87974584-4a620900-ca98-11ea-8dfd-07da7c086ba7.gif)


### Idealized Tetrahedral (≈70µm)

![(2) Idealized Tetrahedral](https://user-images.githubusercontent.com/7799699/87972655-484a7b00-ca95-11ea-8214-244e0fbb817b.gif) 

###### Nodal voltage time-series are visualized after exciting a resistor-network model (Admittance Method, AM) with compartmental currents from a neural network model implanted in NEURON [yale]. The color of each ball encodes charge polarity while the size denotes voltage amplitude. These results differ in principle geometry of the meshed models with grid-based hexahedral and unstructured (equifacial ideal) tetrahedral elements on the left and right, respectively. These results show similar patterns of field potentials but the tetrahedral mesh has somewhat greater intensity. While LFP estimates are slightly larger in amplitude in tetrahedral models, much of this effect is an illusion created by perspective and non-overlapping node locations in the unstructured mesh.

![(3) Neurospice](https://user-images.githubusercontent.com/7799699/87972759-70d27500-ca95-11ea-8aa2-18725a2fa3dc.gif)

###### This visualization presents the source data used to develop the models presented in this manuscript. Each ball represents the location of a current source generated by a neuronal compartment. The color and each ball encodes the charge polarity and the size denotes the normalized amplitude of the voltage each ball contributes to the potential as measured at the recording electrode. This normalization was performed for each time-step independently using the line-source equation [holt & Koch]. A feature of note in this data set is that the spatial distribution of important current sources varies dramatically from beginning to end of the simulated behavior.


![(4) Neurospice quadrant](https://user-images.githubusercontent.com/7799699/87974382-fd7e3280-ca97-11ea-8557-531b1c5ef9d0.jpg)

![(5) Neurospice ](https://user-images.githubusercontent.com/7799699/87973143-11c13000-ca96-11ea-94b2-b89ff60b10c7.jpg)

![(6) Neurospice](https://user-images.githubusercontent.com/7799699/87973438-8300e300-ca96-11ea-97a8-36f51d4cc912.jpg)

![(7) Neurospice](https://user-images.githubusercontent.com/7799699/87973730-f6a2f000-ca96-11ea-9cd0-8b7c90591e22.jpg)

![(8) Neurospice Diagram](https://user-images.githubusercontent.com/7799699/87973796-10dcce00-ca97-11ea-9fa2-2900ab6f7f50.jpg)

Example Script:

```
#!/usr/bin/env python

import os
import neurospice
import numpy as np
from neurospice.MeshVisualizer import VisualizeTetMesh, VisualizeHexMesh
from neurospice.BoundingBoxBuilder import BuildBoundingBox
from neurospice.CurrentSourceManager import ShiftCurrents, SplitCurrents
from neurospice.BasicGeometries import CubicNodes, UnstructuredNodes
from neurospice.SolutionBuilder import *
from neurospice.ConvenientTools import helpme,timeit
from neurospice.CubicModelPlotter import PlotModelSimulation, constant_camera_view, PlotVoltageTraces
import time
import pickle
import zipfile
try:
    import zlib
    compression = zipfile.ZIP_DEFLATED
except:
    compression = zipfile.ZIP_STORED


class DummyBB_RecLocs():
	def __init__(self):
		self.recommended_shift = [191.40938930493894, -181.48026214722756, 100.08796093020896]
		self.recommended_dimensions = [3498.4256946710384, 1911.1963952599187, 714.9837584505374]
		lower = np.array(self.recommended_shift)
		upper = np.array(self.recommended_dimensions)+lower
		self.xs = np.arange(lower[0]+20,upper[0]+20,75.0)
		self.ys = np.arange(lower[1]+5,upper[1]+5,75.0)
		self.locs_shape = (len(self.xs),len(self.ys),1)
		self.RecLocs = []
		for x in self.xs:
			for y in self.ys:
				self.RecLocs.append((x,y,120.1))

def build_and_solve_LFPs(recording_points=[[935.84, 243.9, 120.1]],unique_name=None,datadir=None,vol_scalar=1.42,node_count=1001,current_handling_algo='shift',geometry='structured_hex',visualize=False,threads=3,write_video=False):
	print('\n')
	if datadir ==None:
		datadir==os.getcwd()
	
	try:
		boundingbox = BuildBoundingBox(datadir,1.42) #(datadir,padding_proportion)
	except:
		print('Either the data cannot be found in the directory provided or the data is formated incorrectly. Terminating process')
		boundingbox = DummyBB_RecLocs()	
	
	
	if geometry == 'structured_hex' or geometry=='structured_tet':
		nodes = CubicNodes(boundingbox.recommended_dimensions,node_count,boundingbox.recommended_shift)
	
	else:
		nodes = UnstructuredNodes(boundingbox.recommended_dimensions,node_count,boundingbox.recommended_shift)
	
	
	with open(unique_name+'.nodes','wb') as f:
		pickle.dump(nodes,f)
	
	if visualize and geometry == 'structured_hex':
		msh = VisualizeHexMesh(nodes.xs,nodes.ys,nodes.zs)
	if visualize and geometry != 'structured_hex':
		msh = VisualizeTetMesh(nodes.xs,nodes.ys,nodes.zs,3200) #plot vtk mesh and output image : default args plot=True, save=True

	if current_handling_algo == 'split':
		if geometry != 'structured_hex':
			sources = SplitCurrents(list(zip(nodes.xs,nodes.ys,nodes.zs)),datadir,100,geo='tet',numthreads=threads)
		else:
			sources = SplitCurrents(list(zip(nodes.xs,nodes.ys,nodes.zs)),datadir,100,geo='hex',numthreads=threads)
	else:
		sources = ShiftCurrents(list(zip(nodes.xs,nodes.ys,nodes.zs)),datadir,100,numthreads=threads) #(AM-circuit nodes, cellInfo directory)
	
	with open(unique_name+'_sources.pickle','wb') as f:
		pickle.dump(sources,f)
	
	stime = time.time()
	if geometry != 'structured_hex':
		solver = BuildSolution(nodes,sources,range(200,260),geo='tetrahedral',numthreads = 4)
	else:
		solver = BuildSolution(nodes,sources,range(200,260),geo='hexahedral',numthreads = 4)
	solver.info+='\naverage time to solve circuit time-step (corrected for numthreads): '+str((3*(time.time()-stime))/(60.0*60.0))
	try:
		with open('solverinfo.log','a') as f:
			f.write(unique_name+' '+str(node_count)+' '+'target node count\n')
			f.write(solver.info+'\n\n')
	except:
		with open('solverinfo.log','w') as f:
			f.write(unique_name+' '+str(node_count)+' '+'target node count\n')
			f.write(solver.info+'\n\n')
	
	if write_video:
		plotter = PlotModelSimulation(nodes,range(200,260),os.getcwd())
	
	if geometry != 'structured_hex':
		plotter = PlotVoltageTraces(nodes,geo='tet',name=unique_name) 
	else:
		plotter = PlotVoltageTraces(nodes,geo='hex',name=unique_name) 
	
	plotter.interpolated_recording_to_csv(points=dummy.RecLocs)
	
	flist = [fname for fname in os.listdir() if '_solution' in fname or unique_name in fname]
	zf = zipfile.ZipFile(unique_name+".zip", "w")
	for f in flist:
		zf.write(f,compress_type=compression)
	
	zf.close()




if __name__ == '__main__':
	data_dir = '/run/media/AM-NEURON_LFP_Estimation_Data/Raw'
	nthreads = 4
	recloc = [[935.84, 243.9, 120.1]]
	for ncount in range(6501,8501,2000):
		build_and_solve_LFPs(recording_points=recloc,unique_name='shift_struc_hex_'+str(ncount),datadir=data_dir,node_count=ncount,current_handling_algo='shift',geometry='structured_hex',threads=nthreads)
		
build_and_solve_LFPs(recording_points=recloc,unique_name='shift_unstruc_tet_'+str(ncount),datadir=data_dir,node_count=ncount,current_handling_algo='shift',geometry='unstructured_tet',threads=nthreads)
				build_and_solve_LFPs(recording_point=recloc,unique_name='split_struc_hex_'+str(ncount),datadir=data_dir,node_count=ncount,current_handling_algo='split',geometry='structured_hex',threads=nthreads)
		
build_and_solve_LFPs(recording_point=recloc,unique_name='split_unstruc_tet_'+str(ncount),datadir=data_dir,node_count=ncount,current_handling_algo='split',geometry='unstructured_tet',threads=nthreads)

```
