Igor Pro module for reading and writing NeurodataWithoutBorder files
--------------------------------------------------------------------

This modules allows to easily write and read valid `NeurodataWithoutBorder <https://nwb.org>`__ style HDF5
files. It encapsulates most parts of the specification in easy to use functions.

Main features:

- Read and write NWB compliant files (specification version 1.0.5)
- Compatible with Igor Pro 7 on Windows/MacOSX
- Requires the stock HDF5 XOP only

Example of writing into NWB
^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: igorpro
   :linenos:

    Function NWBWriterExample()

      variable fileID
      string contents, device

      // Open a dialog for selecting an HDF5 file name
      HDF5CreateFile fileID as ""

      // If you open an existing NWB file to append to, use the following command
      // to add an modification time entry
      // IPNWB#AddModificationTimeEntry(locationID)

      // fill gi/ti/si with appropriate data for your lab and experiment
      // if you don't care about that info just pass the initialized structures
      STRUCT IPNWB#GeneralInfo gi
      STRUCT IPNWB#ToplevelInfo ti
      STRUCT IPNWB#SubjectInfo si

      // takes care of initializing
      IPNWB#InitToplevelInfo(ti)
      IPNWB#InitGeneralInfo(gi)
      IPNWB#InitSubjectInfo(si)

      IPNWB#CreateCommonGroups(fileID, toplevelInfo=ti, generalInfo=gi, subjectInfo=si)

      // 1D waves from your measurement program
      // we use fake data here
      Make/FREE AD = (sin(p) + cos(p/10)) * enoise(0.1)

      // write AD data to the file
      STRUCT IPNWB#WriteChannelParams params
      IPNWB#InitWriteChannelParams(params)

      params.device          = "My Hardware"
      params.clampMode       = 0 // 0 for V_CLAMP_MODE 1 for I_CLAMP_MODE
      params.channelSuffix   = ""
      params.sweep           = 123
      params.electrodeNumber = 1
      params.electrodeName   = "Nose of the mouse"
      params.stimset         = "My fancy sine curve"
      params.channelType     = 0 // @see IPNWB_ChannelTypes
      WAVE params.data       = AD

      device = "My selfbuilt DAC"

      IPNWB#CreateIntraCellularEphys(fileID)
      sprintf contents, "Electrode %d", params.ElectrodeNumber
      IPNWB#AddElectrode(fileID, params.electrodeName, contents, device)

      // calculate the timepoint of the first wave point relative to the session_start_time
      params.startingTime  = NumberByKeY("MODTIME", WaveInfo(AD, 0)) - date2secs(-1, -1, -1) // last time the wave was modified (UTC)
      params.startingTime -= ti.session_start_time // relative to the start of the session
      params.startingTime -= DimSize(AD, 0) / 1000 // we want the timestamp of the beginning of the measurement, assumes "ms" as wave units

      IPNWB#AddDevice(fileID, "Device name", "My hardware specs")

      STRUCT IPNWB#TimeSeriesProperties tsp
      IPNWB#InitTimeSeriesProperties(tsp, params.channelType, params.clampMode)

      // all values not added are written into the missing_fields dataset
      IPNWB#AddProperty(tsp, "capacitance_fast", 1.0)
      IPNWB#AddProperty(tsp, "capacitance_slow", 1.0)

      // setting chunkedLayout to zero makes writing faster but increases the final filesize
      IPNWB#WriteSingleChannel(fileID, "/acquisition/timeseries", params, tsp, chunkedLayout=0)

      // write DA, stimulus presentation and stimulus template accordingly
      // ...

      // close file
      HDF5CloseFile fileID
    End

Example reader code
^^^^^^^^^^^^^^^^^^^

FIXME

NWB file format description
^^^^^^^^^^^^^^^^^^^^^^^^^^^

- Datasets which originate from Igor Pro waves have the special
  attributes IGORWaveScaling, IGORWaveType, IGORWaveUnits,
  IGORWaveNote. These attributes allow easy and convenient loading of
  the data into Igor Pro back.
- For AD/DA/TTL groups the naming scheme is
  data\_\ ``XXXXX``\ \_[AD/DA/TTL]\ ``suffix`` where ``XXXXX`` is a
  running number incremented for every sweep ``suffix`` the channel number
  (TTL channels: plus TTL line).
- For I=0 clamp mode neither the DA data nor the stimset is saved.
- Some entries in the following tree are specific to MIES, these are marked
  as custom entries. Users running MIES are encouraged to use the same NWB
  layout and extensions.

The following tree describes the currently supported NWB layout
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: none
   :linenos:

   acquisition:
        timeseries: (empty if no acquired data is saved)
            data_XXXXX_ADY:
                    stimulus_description : custom entry, name of the stimset
                    data                 : 1D dataset with attributes unit, conversion and resolution
                    electrode_name       : Name of the electrode headstage, more info in /general/intracellular_ephys/electrode_name
                    gain                 : scaling factor
                    num_samples          : Number of rows in data
                    starting_time        : relative to /session_start_time with attributes rate and unit
                    For Voltage Clamp (Missing entries are mentioned in missing_fields):
                    capacitance_fast
                    capacitance_slow
                    resistance_comp_bandwidth
                    resistance_comp_correction
                    resistance_comp_prediction
                    whole_cell_capacitance_comp
                    whole_cell_series_resistance_comp

                    For Current Clamp (Missing entries are mentioned in missing_fields):
                    bias_current
                    bridge_balance
                    capacitance_compensation

                    description    : Unused
                    source         : Human readable description of the source of the data
                    comment        : User comment for the sweep
                    missing_fields : Entries missing for voltage clamp/current clamp data
                    ancestry       : Class hierarchy defined by NWB spec, important members are
                                     CurrentClampSeries, IZeroClampSeries and VoltageClampSeries
                    neurodata_type : TimeSeries

    stimulus:
        presentation: (empty if no acquired data is saved)
            data_XXXXX_DA_Y: DA data as sent to the neuron, including delays, scaling, initial TP, etc.
                    data           : 1D dataset
                    electrode_name : Name of the electrode headstage, more info in /general/intracellular_ephys/electrode_name
                    gain           :
                    num_samples    : Number of rows in data
                    starting_time  : relative to /session_start_time with attributes rate and unit
                    description    : Unused
                    source         : Human readable description of the source of the data
                    ancestry       : Class hierarchy defined by NWB spec, important members are
                                     CurrentClampStimulusSeries and VoltageClampStimulusSeries
                    neurodata_type : TimeSeries

        template: unused

    general:
        devices: (empty if no acquired data is saved)
            device_XXX: Name of the DA_ephys device, something like "Harvard Bioscience ITC 18USB"
            intracellular_ephys:
                    electrode_XXX: (XXX can be set by the user via writing into GetCellElectrodeNames())
                        description : Holds the description of the electrode, something like "Headstage 1".
                        device      : Device used to record the data

        labnotebook: custom entry
            XXXX: Name of the device
                numericalKeys   : Numerical labnotebook
                numericalValues : Keys for numerical labnotebook
                textualKeys     : Keys for textual labnotebook
                textualValues   : Textual labnotebook

        testpulse: custom entry
            XXXX: Name of the device
                TPStorage/TPStorage_X: testpulse property waves

        user_comment:
            XXXX: Name of the device
                userComment: All user comments from this session

        generated_by: custom entry
            Nx2 text data array describing the system which created the data. First column is the key, second the value.

        stimsets: custom entry
            XXXXXX_[DA/TTL]_Y_[SegWvType/WP/WPT]: The Wavebuilder parameter waves. These waves will not be available for
                                              "third party stimsets" created outside of MIES.
            XXXXXX_[DA/TTL]_Y: Name of the stimset, referenced from
                             stimulus_description if acquired data is present. Only present if
                             not all parameter waves could be found.
            referenced: All referenced custom waves are stored here in a file-system like group-structure.
                        /general/stimsets/referenced/ relates to root: in the igor Experiment.

    file_create_date    : text array with UTC modification timestamps
    identifier          : SHA256 hash, ensured to be unique
    nwb_version         : NWB specification version
    session_description : unused
    session_start_time  : UTC timestamp defining when the recording session started

    epochs:
        tags: unused

    The following entries are only available if explicitly set by the user:
        data_collection
        experiment_description
        experimenter
        institution
        lab
        notes
        pharmacology
        protocol
        related_publications
        session_id
        slices
        stimulus:
                age
                description
                genotype
                sex
                species
                subject_id
                weight
        surgery
        virus

Online Resources
~~~~~~~~~~~~~~~~

-  https://neurodatawithoutborders.github.io
-  https://crcns.org/NWB