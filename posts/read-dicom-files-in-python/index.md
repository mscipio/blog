<!-- 
.. title: Read DICOM files in Python
.. slug: read-dicom-files-in-python
.. date: 2016-04-18 13:07:55 UTC+02:00
.. tags: dicom; python; pydicom
.. category: python;dicom
.. link: 
.. description: 
.. type: text
-->

## Dataset
Dataset is the main object you will work with directly. Dataset is derived from python’s dict, so it inherits (and overrides some of) the methods of dict. In other words it is a collection of key:value pairs, where the key value is the DICOM (group,element) tag (as a Tag object, described below), and the value is a DataElement instance (also described below).

A dataset could be created directly, but you will usually get one by reading an existing DICOM file:

```python
>>> import pydicom
>>> ds = pydicom.read_file("rtplan.dcm") # (rtplan.dcm is in the testfiles directory)
```

You can display the entire dataset by simply printing its string (str or repr) value:

```python
>>> ds
(0008, 0012) Instance Creation Date              DA: '20030903'
(0008, 0013) Instance Creation Time              TM: '150031'
(0008, 0016) SOP Class UID                       UI: RT Plan Storage
(0008, 0018) SOP Instance UID                    UI: 1.2.777.777.77.7.7777.7777.20030903150023
(0008, 0020) Study Date                          DA: '20030716'
(0008, 0030) Study Time                          TM: '153557'
(0008, 0050) Accession Number                    SH: ''
(0008, 0060) Modality                            CS: 'RTPLAN'
```

You can access specific data elements by name (DICOM ‘keyword’) or by DICOM tag number:

```python
>>> ds.PatientName
'Last^First^mid^pre'
>>> ds[0x10,0x10].value
'Last^First^mid^pre'
```

In the latter case (using the tag number directly) a DataElement instance is returned, so the .value must be used to get the value.

You can also set values by name (DICOM keyword) or tag number:

```python
>>> ds.PatientID = "12345"
>>> ds.SeriesNumber = 5
>>> ds[0x10,0x10].value = 'Test'
```

The use of names is possible because Pydicom intercepts requests for member variables, and checks if they are in the DICOM dictionary. It translates the keyword to a (group,element) number and returns the corresponding value for that key if it exists.

DICOM Sequences are turned into python list s. Items in the sequence are referenced by number, beginning at index 0 as per python convention.:

```python
>>> ds.BeamSequence[0].BeamName
'Field 1'
```

Using DICOM keywords is the recommended way to access data elements, but you can also use the tag numbers directly, such as:

```python
>>> # Same thing with tag numbers:
>>> ds[0x300a,0xb0][0][0x300a,0xc2].value
'Field 1'
>>> # yet another way, using another variable
>>> beam1=ds[0x300a,0xb0][0]
>>> beam1.BeamName, beam1[0x300a,0xc2].value
('Field 1', 'Field 1')
```

If you don’t remember or know the exact tag name, Dataset provides a handy dir() method, useful during interactive sessions at the python prompt:

```python
>>> ds.dir("pat")
['PatientBirthDate', 'PatientID', 'PatientName', 'PatientSetupSequence', 'PatientSex']
```

dir will return any DICOM tag names in the dataset that have the specified string anywhere in the name (case insensitive).

>**NOTE:**
>Calling dir with no string will list all tag names available in the dataset.
>You can also see all the names that pydicom knows about by viewing the _dicom_dict.py file. You could >modify that file to add tags that pydicom doesn’t already know about.

Under the hood, Dataset stores a DataElement object for each item, but when accessed by name (e.g. ds.PatientName) only the value of that DataElement is returned. If you need the whole DataElement (see the DataElement class discussion), you can use Dataset’s data_element() method or access the item using the tag number:

```python
>>> data_element = ds.data_element("PatientsName")  # or data_element = ds[0x10,0x10]
>>> data_element.VR, data_element.value
('PN', 'Last^First^mid^pre')
```

To check for the existence of a particular tag before using it, use the in keyword:
```python
>>> "PatientName" in ds
True
```
To remove a data element from the dataset, use del:

```python
>>> del ds.SoftwareVersions   # or del ds[0x0018, 0x1020]
```

To work with pixel data, the raw bytes are available through the usual tag:

```python
>>> pixel_bytes = ds.PixelData
```

but to work with them in a more intelligent way, use pixel_array (requires the NumPy library):

```python
>>> pix = ds.pixel_array
```

For more details, see a future post [Working with Pixel Data.]

## DataElement
The DataElement class is not usually used directly in user code, but is used extensively by Dataset. DataElement is a simple object which stores the following things:

* **tag** – a DICOM tag (as a Tag object)
* **VR** – DICOM value representation – various number and string formats, etc
* **VM** – value multiplicity. This is 1 for most DICOM tags, but can be multiple, e.g. for coordinates. You do not have to specify this, the DataElement class keeps track of it based on value.
* **value** – the actual value. A regular value like a number or string (or list of them), or a Sequence.

## Tag
The Tag class is derived from python’s *int*, so in effect, it is just a number with some extra behaviour:

* Tag enforces that the DICOM tag fits in the expected 4-byte (group,element)
* A Tag instance can be created from an int or from a tuple containing the (group,element) separately:
 
```python
>>> from pydicom.tag import Tag
>>> t1=Tag(0x00100010) # all of these are equivalent
>>> t2=Tag(0x10,0x10)
>>> t3=Tag((0x10, 0x10))
>>> t1
(0010, 0010)
>>> t1==t2, t1==t3
(True, True)
```

* Tag has properties group and element (or elem) to return the group and element portions
* The is_private property checks whether the tag represents a private tag (i.e. if group number is odd).

## Sequence
Sequence is derived from python’s *list*. The only added functionality is to **make string representations prettier**. Otherwise all the usual methods of list like item selection, append, etc. are available.
