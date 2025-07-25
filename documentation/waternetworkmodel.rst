.. raw:: latex

    \clearpage

.. doctest::
    :hide:
	
    >>> import numpy as np
    >>> import pandas as pd
    >>> try:
    ...    import geopandas as gpd
    ... except ModuleNotFoundError:
    ...    gpd = None
    >>> import matplotlib as mpl
    >>> try:
    ...     mpl.use('Agg')
    ... except:
    ...     pass
	
Water network model
======================================

The water network model includes 
junctions, tanks, reservoirs, pipes, pumps, valves, 
patterns, 
curves,
controls, 
sources,
simulation options,
and node coordinates.
Water network models can be built from scratch or built directly from an EPANET INP file.
Sections of the EPANET INP file that are not compatible with WNTR are described in :ref:`limitations`.  
For more information on the water network model, see 
:class:`~wntr.network.model.WaterNetworkModel` in the API documentation.

Build a model from an EPANET INP file
-------------------------------------

A water network model can be created directly from EPANET INP files using EPANET 2.00.12 or 2.2.0 format.  
The following example builds a water network model from an EPANET INP file.

.. doctest::

    >>> import wntr
	
    >>> wn = wntr.network.WaterNetworkModel('networks/Net3.inp') # doctest: +SKIP

A water network model can also be created using EPANET INP files from the model library, see :ref:`model_library` for more details.
For example, the following loads 'Net3.inp' from the model library.

.. doctest::

    >>> wn = wntr.network.WaterNetworkModel('Net3')
	
.. note:: 
  Unless otherwise noted, examples in the WNTR documentation use Net3.inp to build the
  water network model, named ``wn``.
 
Add elements
------------------

The water network model contains methods to add 
junctions, tanks, reservoirs, 
pipes, pumps, valves,
patterns, curves, sources, and controls.
When an element is added to the model, it is added to the model's registry.
Within the registry, junctions, tanks, and reservoirs share a namespace (e.g., those elements cannot share names) and pipes, pumps, and valves share a namespace.

For each method that adds an element to the model, there is a related object.  For example, the 
:class:`~wntr.network.model.WaterNetworkModel.add_junction` method adds a 
:class:`~wntr.network.elements.Junction` object to the model.
Generally, the object is not added to the model directly.

The example below adds a junction and pipe to a water network model.

.. doctest::

    >>> wn.add_junction('new_junction', base_demand=10, demand_pattern='1', elevation=10, 
    ...     coordinates=(6, 25))
    >>> wn.add_pipe('new_pipe', start_node_name='new_junction', end_node_name='101', 
    ...     length=10, diameter=0.5, roughness=100, minor_loss=0)
			
Remove elements
------------------

The water network model registry tracks when elements are used by other elements in the model. 
An element can only be removed if all elements that rely on it are removed or modified. 
For example, if a valve is used in a control, the valve cannot be removed until the control is removed or modified. 
Similarly, a node cannot be removed until the pipes connected to that node are removed.  
The following example removes a link and node from the model. 
If the element being removed is used by another element, an error message is printed to the screen and the element is not removed.

.. doctest::

    >>> wn.remove_link('new_pipe')
    >>> wn.remove_node('new_junction')

Modify options
--------------------------

Water network model options are divided into the following categories:
time, hydraulics, quality, solver, results, graphics, and energy. 
The following example returns model options, which all have default values,
and then modifies the simulation duration.

.. doctest::

    >>> wn.options # doctest: +SKIP
    Time options:
      duration            : 604800              
      hydraulic_timestep  : 900                 
      quality_timestep    : 900                 
      rule_timestep       : 360.0               
      pattern_timestep    : 3600
    ...
    >>> wn.options.time.duration = 10*3600
	
Modify element attributes
---------------------------------------

To modify element attributes, the element object is first obtained using the
:class:`~wntr.network.model.WaterNetworkModel.get_node` or 
:class:`~wntr.network.model.WaterNetworkModel.get_link` methods.
The following example changes junction elevation, pipe diameter, and size for a constant diameter tank.

.. doctest::

    >>> junction = wn.get_node('121')
    >>> junction.elevation = 5
    >>> pipe = wn.get_link('122')
    >>> pipe.diameter = pipe.diameter*0.5
    >>> tank = wn.get_node('1')
    >>> tank.diameter = tank.diameter*1.1

The following shows how to add an additional demand to the junction 121.

.. doctest::

    >>> print(junction.demand_timeseries_list)  # doctest: +SKIP
    <Demands: [<TimeSeries: base_value=0.002626444876132, pattern_name='1', category='None'>]> 
    
    >>> junction.add_demand(base=1.0, pattern_name='1')
    >>> print(junction.demand_timeseries_list)  # doctest: +SKIP
    <Demands: [<TimeSeries: base_value=0.002626444876132, pattern_name='1', category='None'>, <TimeSeries: base_value=1.0, pattern_name='1', category='None'>]>

To remove the demand, use the Python ``del`` as with an array element.

.. doctest::

    >>> del junction.demand_timeseries_list[1]
    >>> print(junction.demand_timeseries_list)
    <Demands: [<TimeSeries: base_value=0.002626444876132, pattern_name='1', category='None'>]>


Modify time series
-------------------------------

Several network attributes are stored as a time series, including 
junction demand, reservoir head, and pump speed. 
A time series contains a base value, a pattern, and a category.
Time series are added to the water network model when the junction, 
reservoir, or pump is added.
Since junctions can 
have multiple demands, junction demands are stored as a list of time series.
The following examples modify time series.

Change reservoir supply:

.. doctest::

    >>> reservoir = wn.get_node('River')
    >>> reservoir.head_timeseries.base_value = reservoir.head_timeseries.base_value*0.9

Change junction demand base value:

.. doctest::

    >>> junction = wn.get_node('121')
    >>> junction.demand_timeseries_list[0].base_value = 0.005
	
Add a new demand time series to the junction:

.. doctest::

    >>> pat = wn.get_pattern('3')
    >>> junction.demand_timeseries_list.append((0.001, pat))


Add custom element attributes
---------------------------------------

New attributes can be added to model elements simply by defining a new attribute 
name and value. These attributes can be used in custom analysis and graphics.
While this is similar to using external datasets directly, there can be benefits 
to adding custom attributes to model objects.

The following example uses a dataset that defines pipe material as 
polyvinyl chloride (PVC), cast iron, steel, or high-density polyethylene (HDPE). 
The dataset is indexed by pipe name.  The first 10 lines of the dataset are shown below.

.. doctest::
    :hide:

    >>> np.random.seed(6789)
    >>> material = pd.Series()
    >>> for name, pipe in wn.pipes():
    ...     val = np.random.choice(['PVC', 'Cast iron', 'Steel', 'HDPE'], 1, [0.45, 0.2, 0.1, 0.25])[0]
    ...     material[name] = val

.. doctest::

    >>> print(material.head(10))
    20         Steel
    40     Cast iron
    50          HDPE
    60           PVC
    101    Cast iron
    103          PVC
    105        Steel
    107        Steel
    109    Cast iron
    111        Steel
    dtype: object
	
The data can be used to create a custom `material` attribute for each pipe object.

.. doctest::

    >>> for name, pipe in wn.pipes():
    ...     pipe.material = material[name]

The custom attribute can be used in analysis in several ways. 
For example, the following example closes the pipe if pipe material is 'Steel.'

    >>> for name, pipe in wn.pipes():
    ...     if pipe.material == 'Steel':
    ...         pipe.initial_status = 'Closed'

A complete list of custom attributes can also be obtained using a query, 
as shown below.

.. doctest::

    >>> material = wn.query_link_attribute('material')

    
Iterate over elements
-------------------------

Iterators are available for 
junctions, tanks, reservoirs,
pipes, pumps, and valves.  
Each iterator returns the element's name and the element's object.
The following example iterates over all pipes to 
modify pipe diameter.

.. doctest::

    >>> for pipe_name, pipe in wn.pipes():
    ...     pipe.diameter = pipe.diameter*0.9

Get element names and counts
-----------------------------------

Several methods are available to return a list of element names and the
number of elements, as shown in the
example below.  The list of element names can be used as an iterator, especially in cases 
where the element object is not needed. 

.. doctest::

    >>> node_names = wn.node_name_list
    >>> num_nodes = wn.num_nodes
    >>> wn.describe(level=0) # doctest: +SKIP
    {'Nodes': 97, 'Links': 119, 'Patterns': 5, 'Curves': 2, 'Sources': 0, 'Controls': 18}
	 
Query element attributes
---------------------------

The water network model contains methods to query node and link attributes.  These methods can 
return attributes for all nodes or links, or for a subset using arguments that specify a node or link type 
(i.e., junction or pipe), or by specifying a threshold (i.e., >= 10 m).  
The query methods return a pandas Series with the element name and value.
The following example returns node elevation, junction elevation, and junction elevations greater than 10 m (using a
NumPy operator).

.. doctest::

    >>> import numpy as np
    
    >>> node_elevation = wn.query_node_attribute('elevation')
    >>> junction_elevation = wn.query_node_attribute('elevation', 
    ...     node_type=wntr.network.model.Junction)
    >>> junction_elevation_10 = wn.query_node_attribute('elevation', np.greater_equal, 
    ...     10, node_type=wntr.network.model.Junction)
	
In a similar manner, link attributes can be queried, as shown below.

.. doctest::

    >>> link_length = wn.query_link_attribute('length', np.less, 50) 

Reset initial conditions
-----------------------------

When using the same water network model to run multiple simulations using the WNTRSimulator, initial conditions need to be reset between simulations.  
Initial conditions include simulation time, tank head, reservoir head, pipe status, pump status, and valve status.
When using the EpanetSimulator, this step is not needed since EPANET starts at the initial conditions each time it is run.

.. doctest::

    >>> wn.reset_initial_values()

Build a model from scratch
---------------------------------

A water network model can also be created from scratch by adding elements to an empty model.  Elements 
must be added before they are used in a simulation.  For example, demand patterns are added to the model before they are 
used within a junction. The section below includes additional information on adding elements to a 
water network model.
 
.. doctest::

    >>> wn = wntr.network.WaterNetworkModel()
    >>> wn.add_pattern('pat1', [1])
    >>> wn.add_pattern('pat2', [1,2,3,4,5,6,7,8,9,10])
    >>> wn.add_junction('node1', base_demand=0.01, demand_pattern='pat1', elevation=100, 
    ...     coordinates=(1,2))
    >>> wn.add_junction('node2', base_demand=0.02, demand_pattern='pat2', elevation=50, 
    ...     coordinates=(1,3))
    >>> wn.add_pipe('pipe1', 'node1', 'node2', length=304.8, diameter=0.3048, 
    ...    roughness=100, minor_loss=0.0, initial_status='OPEN')
    >>> wn.add_reservoir('res', base_head=125, head_pattern='pat1', coordinates=(0,2))
    >>> wn.add_pipe('pipe2', 'node1', 'res', length=100, diameter=0.3048, roughness=100, 
    ...     minor_loss=0.0, initial_status='OPEN')
    >>> ax = wntr.graphics.plot_network(wn)

.. doctest::
    :hide:

    >>> sim = wntr.sim.EpanetSimulator(wn) # make sure it's a valid model
    >>> results = sim.run_sim()
