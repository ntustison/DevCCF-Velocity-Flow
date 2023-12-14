# The DevCCF velocity flow transformation model

## Description

A multimodal 3D developmental common coordinate framework (DevCCF) spanning 
mouse embryonic day (E) 11.5, E13.5, E15.5, E18.5, and postnatal day (P) 4, 
P14, and P56 with anatomical segmentations defined by a developmental ontology
is described in [Kronman et al.](https://www.biorxiv.org/content/10.1101/2023.09.14.557789v1).
At each age, the DevCCF features undistorted morphologically averaged atlas 
templates created from Magnetic Resonance Imaging and co-registered high-
resolution templates from light sheet fluorescence microscopy. Expert-curated 
3D anatomical segmentations at each age are [also available](). 
As part of this work, we generate a [diffeomorphic velocity flow model](https://en.wikipedia.org/wiki/Large_deformation_diffeomorphic_metric_mapping)
permitting deformations between the above mouse developmental stages and
at any continuous point within the developmental range.  The transformation
model is generated using the publicly available data provided through the
above cited work and using [ANTsX tools](https://github.com/ANTsX).  This
repository provides the scripts to both reproduce and utilize the velocity
flow field.

## Preliminaries

### Code

All data processing uses [ANTsPy](https://github.com/ANTsX/ANTsPy) with 
equivalent calls possible in [ANTsR](https://github.com/ANTsX/ANTsR).
Be sure to [install ANTsPy](https://github.com/ANTsX/ANTsPy#installation)
prior to attempting to reproduce the results below.  To test your installation 
in the context of this work,  please attempt to reproduce a 
[small, self-contained example](https://gist.github.com/ntustison/12a656a5fc2f6f9c4494c88dc09c5621#file-b_3_ants_velocity_flows-md)
illustrating the code and principles used.  Conceptually, this code snippet 
creates a time-parameterized velocity flow model in the range $t=[0,1]$ using 
three 2-D point sets comprising 8 points each representing a rectangle at $t=0.0$, 
a square at $t=0.5$, and a circle at $t=1.0$.  The ANTsPy example should produce the 
following plots:

<p align="middle">
  <img src="https://github.com/ntustison/MouseBrainVelocityFlow/assets/324811/dbc63553-27ad-4130-8bbf-c10cdf8fc893" width="250" />
  <img src="https://github.com/ntustison/MouseBrainVelocityFlow/assets/324811/cd78595b-1e12-47fc-b606-ae4b5012cbd6" width="250" /> 
  <img src="https://github.com/ntustison/MouseBrainVelocityFlow/assets/324811/c7ee9ad6-1f3a-4da4-832e-ba64b1b15f31" width="250" /> 
</p>

<!--
![original_data](https://github.com/ntustison/MouseBrainVelocityFlow/assets/324811/dbc63553-27ad-4130-8bbf-c10cdf8fc893)
![warping_between_endpoints](https://github.com/ntustison/MouseBrainVelocityFlow/assets/324811/cd78595b-1e12-47fc-b606-ae4b5012cbd6)
![warping_to_middle](https://github.com/ntustison/MouseBrainVelocityFlow/assets/324811/c7ee9ad6-1f3a-4da4-832e-ba64b1b15f31)
-->

### Data

To simplify this repository, only the data used to create the velocity flow model is 
available in this repository in Data/DevCCFSimpleSegmentations/.  The label images in
this directory are the simplified annotations comprising 26 common regions (13 per 
hemisphere) across all developmental stages.  The full DevCCF atlas can be downloaded 
from [here](https://kimlab.io/brain-map/DevCCF/).  

## Reproducing the DevCCF Velocity Flow Model

### Part I:  Rigidly register everything to P56

```python

###
#
# First, we register all the input label images to P56 as all the images need to reside 
# in a common post-linearly aligned space.  To do this, we find the common labels 
# between all the developmental stages and then use those to find a rigid transform
# to the P56 template. Save the rigid transforms and warped images.  We also resample
# to (0.05, 0.05, 0.05).
# 

import ants
import os

data_directory = "Data/DevCCFSimpleSegmentations/"
output_directory = "Data/Output/P56RigidTransformData/"

if not os.path.exists(output_directory):
    os.makedirs(output_directory, exist_ok=True)

atlas_ids = tuple(reversed(("E11-5", "E13-5", "E15-5", "E18-5", "P04", "P14", "P56")))

common_label_ids = None
for i in range(len(atlas_ids)):
    print("Finding common label ids for atlas:", atlas_ids[i])
    labels_file = data_directory + atlas_ids[i] + "_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
    labels = ants.image_read(labels_file)
    label_geoms = ants.label_geometry_measures(labels)
    if i == 0: 
        common_label_ids = np.array(label_geoms['Label'])
    else:
        common_label_ids = np.intersect1d(common_label_ids, np.array(label_geoms['Label']))
print("Common label ids:", common_label_ids)      

fixed_labels_file = data_directory + "P56_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
fixed_labels = ants.image_read(fixed_labels_file)
label_geoms = ants.label_geometry_measures(fixed_labels)
fixed_label_ids = np.array(label_geoms['Label'])
for l in range(len(fixed_label_ids)):
    if not np.isin(fixed_label_ids[l], common_label_ids):
        fixed_labels[fixed_labels == fixed_label_ids[l]] = 0
label_geoms = ants.label_geometry_measures(fixed_labels)
fixed_points = np.zeros((label_geoms.shape[0], 3))
fixed_points[:,0] = label_geoms['Centroid_x']
fixed_points[:,1] = label_geoms['Centroid_y']
fixed_points[:,2] = label_geoms['Centroid_z']
for n in range(fixed_points.shape[0]):
    fixed_points[n,:] = ants.transform_index_to_physical_point(fixed_labels, (fixed_points + 0.5).astype(int)[n,:])

for i in range(len(atlas_ids)):
    print("Processing ", atlas_ids[i])
    moving_labels_file = data_directory + atlas_ids[i] + "_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
    moving_labels = ants.image_read(moving_labels_file)
    label_geoms = ants.label_geometry_measures(moving_labels)
    moving_label_ids = np.array(label_geoms['Label'])
    for l in range(len(moving_label_ids)):
        if not np.isin(moving_label_ids[l], common_label_ids):
            moving_labels[moving_labels == moving_label_ids[l]] = 0
    label_geoms = ants.label_geometry_measures(moving_labels)
    moving_points = np.zeros((label_geoms.shape[0], 3))
    moving_points[:,0] = label_geoms['Centroid_x']
    moving_points[:,1] = label_geoms['Centroid_y']
    moving_points[:,2] = label_geoms['Centroid_z']
    for n in range(moving_points.shape[0]):
        moving_points[n,:] = ants.transform_index_to_physical_point(moving_labels, (moving_points + 0.5).astype(int)[n,:])
    xfrm = ants.fit_transform_to_paired_points(moving_points, fixed_points, transform_type='rigid')
    xfrm_file = output_directory + "P56x" + atlas_ids[i] + "_rigid_xfrm.mat"
    ants.write_transform(xfrm, xfrm_file)
    moving_labels_file = data_directory + atlas_ids[i] + "_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
    moving_labels = ants.image_read(moving_labels_file)
    warped_labels = xfrm.apply_to_image(moving_labels, fixed_labels, interpolation='nearestneighbor') 
    warped_labels = ants.resample_image(warped_labels, (0.05. 0.05, 0.05), False, 1)
    warped_labels_file = output_directory + "P56x" + atlas_ids[i] + "_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
    ants.image_write(warped_labels, warped_labels_file)
```

### Part II:  Perform pairwise registrations of the P56-aligned label images 

```python

###
#
# Second, we perform pairwise registration between temporally adjacent atlases.
# We extract separate images from each fixed/moving pair and construct a separate
# MSQ metric to drive the registration.  
# 

import ants
import glob
import os
import numpy as np

os.environ["ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS"] = "4"

data_directory = "Data/Output/P56RigidTransformData/"
output_directory = "Data/Output/PairwiseRegistrations/"

if not os.path.exists(output_directory):
    os.makedirs(output_directory, exist_ok=True)

template_ids = tuple(reversed(("E11-5", "E13-5", "E15-5", "E18-5", "P04", "P14", "P56")))

for i in range(1, len(template_ids)):
    fixed_labels_file = data_directory + "P56x" + template_ids[i-1] + "_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
    moving_labels_file = data_directory + "P56x" + template_ids[i] + "_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
      
    print("Fixed labels: ", fixed_labels_file)
    print("Moving labels: ", moving_labels_file)

    fixed_labels = ants.image_read(fixed_labels_file)
    moving_labels = ants.image_read(moving_labels_file)

    fixed_label_geoms = ants.label_geometry_measures(fixed_labels)
    fixed_label_ids = np.array(fixed_label_geoms['Label'])
    moving_label_geoms = ants.label_geometry_measures(moving_labels)
    moving_label_ids = np.array(moving_label_geoms['Label'])
    
    label_ids = np.intersect1d(moving_label_ids,fixed_label_ids)
    number_of_labels = len(label_ids)
            
    fixed_image = ants.threshold_image(fixed_labels, 0, 0, 0, 1)
    moving_image = ants.threshold_image(moving_labels, 0, 0, 0, 1)

    fixed_single_label_images = list()
    moving_single_label_images = list()
    for j in range(number_of_labels):
        single_label_image = ants.threshold_image(fixed_labels, label_ids[j], label_ids[j], 1, 0)
        single_label_image = ants.smooth_image(single_label_image, 1, False)
        fixed_single_label_images.append(ants.image_clone(single_label_image))
        single_label_image = ants.threshold_image(moving_labels, label_ids[j], label_ids[j], 1, 0)
        single_label_image = ants.smooth_image(single_label_image, 1, False)
        moving_single_label_images.append(ants.image_clone(single_label_image))

    multivariate_extras = list()            
    for j in range(number_of_labels):
        multivariate_extras.append(["MSQ", fixed_single_label_images[j], moving_single_label_images[j], 10.0, 1])

    output_registration_prefix = output_directory + "P56x" + template_ids[i-1] + "x" + template_ids[i] + "_"

    reg = ants.registration(fixed_image, moving_image, type_of_transform="antsRegistrationSyN[s]", 
                            multivariate_extras=multivariate_extras, 
                            outprefix=output_registration_prefix, verbose=True)    
    print("\n\n\n\n")
```

### Part III:  Extract points in P56, propagate to all developmental atlases, build model

```python

import ants
import os
import pandas as pd
import numpy as np
import random

os.environ["ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS"] = "4"

data_directory = "Data/Output/P56RigidTransformData/"
registration_directory = output_directory + "PairwiseRegistrations/"
output_directory = "Data/Output/"

if not os.path.exists(output_directory):
    os.makedirs(output_directory, exist_ok=True)

################################
#
# A couple notes:
#   * We reverse the template id's because we use P56 to define the fixed reference 
#     frame and, therefore, the positive direction of the velocity field.  This isn't 
#     necessary as we could've easily chosen the opposite direction.  
#   * We take the log of the time points to get a more even distribution of the velocity 
#     field samples.  The only implication of this is that one would need to take into 
#     account this transform when actually using the output velocity field to determine 
#     the transform at a specific time.
#   * Extract contour and regional points in P56.  We use the pairwise registrations to 
#     propagate these points to previous time points.
#   * An additional modification to get a better sampling distribution is to simply use 
#     the time point for P28 = 47 and use the P56 template.
#     

template_ids = tuple(reversed(("E11-5", "E13-5", "E15-5", "E18-5", "P04", "P14", "P56")))
time_points = np.flip(-1.0 * np.log(np.array((11.5, 13.5, 15.5, 18.5, 23, 33, 47))))

contour_percentage = 0.1
regional_percenage = 0.01

fixed_labels_file = annotation_directory + "P56xP56_DevCCF_Annotations_20um_symmetric_commonROIs_hemi.nii.gz"
fixed_labels = ants.image_read(fixed_labels_file)

label_geoms = ants.label_geometry_measures(fixed_labels)
label_ids = np.array(label_geoms['Label'])
number_of_labels = len(label_ids)

contour_indices = list()
for i in range(0, number_of_labels + 1):
    if i < number_of_labels:
        print("Extracting contour points from label ", label_ids[i])
        single_label_image = ants.threshold_image(fixed_labels, label_ids[i], label_ids[i], 1, 0)
    else:
        single_label_image = ants.threshold_image(fixed_labels, 0, 0, 0, 1)
    contour_image = single_label_image - ants.iMath_ME(single_label_image, 1)
    single_label_indices = (contour_image.numpy()).nonzero()
    number_of_points_per_label = int(len(single_label_indices[0]) * contour_percentage)
    print("  Number of points: ", number_of_points_per_label)
    random_indices = random.sample(range(len(single_label_indices[0])), number_of_points_per_label)
    if i == 0:
         contour_indices.append(single_label_indices[0][random_indices])
         contour_indices.append(single_label_indices[1][random_indices])
         contour_indices.append(single_label_indices[2][random_indices])
    else:
         contour_indices[0] = np.concatenate([contour_indices[0], single_label_indices[0][random_indices]])
         contour_indices[1] = np.concatenate([contour_indices[1], single_label_indices[1][random_indices]])
         contour_indices[2] = np.concatenate([contour_indices[2], single_label_indices[2][random_indices]])
         
contour_weights = [1] * len(contour_indices[0])

regional_indices = list()
for i in range(0, number_of_labels + 1):
    if i < number_of_labels:
        print("Extracting regional points from label ", label_ids[i])
        single_label_image = ants.threshold_image(fixed_labels, label_ids[i], label_ids[i], 1, 0)
    else:
        single_label_image = ants.threshold_image(fixed_labels, 0, 0, 0, 1)
    single_label_indices = (single_label_image.numpy()).nonzero()
    number_of_points_per_label = int(len(single_label_indices[0]) * regional_percentage)
    print("  Number of points: ", number_of_points_per_label)
    random_indices = random.sample(range(len(single_label_indices[0])), number_of_points_per_label)
    if i == 0:
         regional_indices.append(single_label_indices[0][random_indices])
         regional_indices.append(single_label_indices[1][random_indices])
         regional_indices.append(single_label_indices[2][random_indices])
    else:
         regional_indices[0] = np.concatenate([regional_indices[0], single_label_indices[0][random_indices]])
         regional_indices[1] = np.concatenate([regional_indices[1], single_label_indices[1][random_indices]])
         regional_indices[2] = np.concatenate([regional_indices[2], single_label_indices[2][random_indices]])
         
regional_weights = [0.5] * len(regional_indices[0])

indices = contour_indices
indices[0] = np.concatenate([indices[0], regional_indices[0]])
indices[1] = np.concatenate([indices[1], regional_indices[1]])
indices[2] = np.concatenate([indices[2], regional_indices[2]])
weights = np.concatenate([contour_weights, regional_weights])

print("Number of contour points:  ", str(len(contour_weights)))
print("Number of regional points:  ", str(len(regional_weights)))

points_time0 = np.zeros((len(indices[0]), 3))
for i in range(len(indices[0])):
    index = (indices[0][i], indices[1][i], indices[2][i])
    points_time0[i,:] = ants.transform_index_to_physical_point(fixed_labels, index)

points_time0_df = pd.DataFrame(points_time0, columns = ('x', 'y', 'z'))

point_sets = list()
point_sets.append(points_time0_df) #P56
for i in range(1, len(template_ids)):
    print("Warping points " + str(i))
    source_template_id = template_ids[i-1]        
    target_template_id = template_ids[i]        
    output_registration_prefix = registration_directory + "P56x" + source_template_id + "x" + target_template_id + "_"
    affine = output_registration_prefix + "0GenericAffine.mat"
    warp = output_registration_prefix + "1Warp.nii.gz"
    warped_points = ants.apply_transforms_to_points(3, points=point_sets[i-1], transformlist=[warp, affine])
    point_sets.append(warped_points)

# Write the points to images to see if they match with what's expected.
check_points = False
if check_points:
    for i in range(len(template_ids)):
        print("Checking image " + str(i))
        points_image = ants.make_points_image(point_sets[i].to_numpy(), fixed_labels * 0 + 1, radius=1)
        output_prefix = output_directory + "P56x" + template_ids[i] + "_"
        ants.image_write(points_image, output_prefix + "points_image.nii.gz")

for i in range(len(point_sets)):
    point_sets[i] = point_sets[i].to_numpy()

# Normalize time points to the range [0, 1]

normalized_time_points = (time_points - time_points[0]) / (time_points[-1] - time_points[0])

initial_velocity_field = None
velocity_field_file = output_directory + "/DevCCF_velocity_flow.nii.gz"
if os.path.exists(velocity_field_file):
    initial_velocity_field = ants.image_read(velocity_field_file)

# We could simply set the total number of iterations (i.e., "number_of_compositions")
# to 10 * 20 but just so we could check the progress, we run the optimization for 10
# iterations and then write the velocity field to disk and use it as the initial 
# velocity field for subsequent iterations.

for i in range(20):
    print("Iteration " + str(i))
    tv = ants.fit_time_varying_transform_to_point_sets(point_sets, time_points=normalized_time_points,
        displacement_weights=weights,
        initial_velocity_field=initial_velocity_field,
        number_of_time_steps=11, domain_image=fixed_labels,
        number_of_fitting_levels=4, mesh_size=4, number_of_compositions=10,
        convergence_threshold=0.0, composition_step_size=0.2,
        number_of_integration_steps=10,
        rasterize_points=False, verbose=True)
    initial_velocity_field = ants.image_clone(tv['velocity_field'])
    ants.image_write(initial_velocity_field, output_directory + "velocity_field.nii.gz")
    print("\n\n\n\n\n\n")
```

## Using the DevCCF Flow Model


```python
# Example:  warp every template to every other template

import ants
import numpy as np
import math

atlas_ids = tuple(reversed(("E11-5", "E13-5", "E15-5", "E18-5", "P04", "P14", "P56")))
time_points = np.flip(-1.0 * np.log(np.array((11.5, 13.5, 15.5, 18.5, 23, 33, 47))))
normalized_time_points = (time_points - time_points[0]) / (time_points[-1] - time_points[0])

velocity_field = ants.image_read("DevCCF_velocity_flow.nii.gz")

# Read template files.
# template_files = list()
# for i in range(len(atlas_ids)):
#      fa_template_files.append(glob.glob(atlas_ids[i] + "*.nii.gz")[0])

for i in range(len(atlas_ids)):
    for j in range(len(atlas_ids)):
        print("Warping ", atlas_ids[j], "to", atlas_ids[i])
        reference_template = ants.image_read(template_files[i])
        moving_template = ants.image_read(template_files[j])
        displacement_field = ants.integrate_velocity_field(velocity_field,
                                                           normalized_time_points[i],
                                                           normalized_time_points[j], 10)
        displacement_field_xfrm = ants.transform_from_displacement_field(displacement_field)
        warped_template = displacement_field_xfrm.apply_to_image(moving_template,
                                                                 interpolation="linear")
```

```python
# Example:  warp P56 in a continuous manner from identity to E11.5

import ants
import numpy as np
import math

velocity_field = ants.image_read("DevCCF_flow_model.nii.gz")
P56 = ants.image_read("P56.nii.gz")  

# We discretize the time domain into 50 intervals.
time_points = np.flip(-1.0 * np.log(np.linspace(11.5, 47, 50)))
normalized_time_points = (time_points - time_points[0]) / (time_points[-1] - time_points[0])

for i in range(len(normalized_time_points)):
    t = normalized_time_points[i]
    displacement_field = ants.integrate_velocity_field(velocity_field, t, 0.0, 10)
    displacement_field_xfrm = ants.transform_from_displacement_field(displacement_field)
    P56warped = displacement_field_xfrm.apply_to_image(P56, interpolation="linear")
```


