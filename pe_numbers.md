#!/usr/local/bin/bash

# Below is a template showing our method to count the number of vertices (positive elements) on a brain surface. 
# Explanation of the work of these commands and how to run a script is beyond the scope of this work. 
# To understand the use of the commands and where to enter the paths to your files and output please execute the command alone in a shell for further explanation.
# To know more about the commands please visit the HCP workbench website https://www.humanconnectome.org/software/workbench-command or visit the HCP-Users Google group https://groups.google.com/a/humanconnectome.org/g/hcp-users
# In this template, cope2 refers to the STORY and the cope4 to the STORY-MATH. 

DATA=path_to_the_main_folder

RESULTS=$DATA/results # This folder will contain later all results' texts and parcellations

HCP=$DATA/hcp_files # This folder already contains necessary files from the HCP, e.g., surfaces, MMP atals, etc.

MMP_ATLAS=Q1-Q6_RelatedValidation210.CorticalAreas_dil_Final_Final_Areas_Group_Colors.32k_fs_LR.dlabel.nii # This is CIFTI file of the MMP parcellation file. The name is too long, better to use a variable instead.

Zero=0 

# You will need two text files, one having list of the names of your subjects, and the other is a list of the name of the MMP regions without L_ or R_

# This script assumes you already have a folder for each subject in the $DATA folder. Each subject's folder shall contain two folders, the COPE2 and COPE4 with the z-stat map inside them, the "zstat1.dtseries.nii"

for sub in $(cat $DATA/subj_list.txt); do
    for cope in COPE2 COPE4; do

# First is to create a surface mask with positive BOLD signals only, let us call it posmask.

        echo "Creating positive BOLD mask for ${sub}.${cope}"
        
        wb_command -cifti-math \
            "fmri > 0" \
            $DATA/${sub}/${cope}/${sub}.${cope}.posmask.dtseries.nii \
            -var fmri $DATA/${sub}/${cope}/zstat1.dtseries.nii

# Use the posmask to create a dense map with the positive BOLD, let us call it positive

        POSMASK=$DATA/${sub}/${cope}/${sub}.${cope}.posmask.dtseries.nii

        echo "Creating positive BOLD image for ${sub}.${cope}"

        wb_command -cifti-math \
            "fmri * pos" \
            $DATA/${sub}/${cope}/${sub}.${cope}.pos.dtseries.nii \
            -var fmri $DATA/${sub}/${cope}/zstat1.dtseries.nii \
            -var pos $POSMASK

        POSITIVE=$DATA/${sub}/${cope}/${sub}.${cope}.pos.dtseries.nii

# Next is to measure the mean z-stat (AVGZ) of the "positive" file (the threshold for each subject).

        SUMZ=$(wb_command -cifti-stats $POSITIVE -reduce SUM)
        # This is the summation value of the activated elements z-stat within the ROI
                
        COUNTZ=$(wb_command -cifti-stats $POSITIVE -reduce COUNT_NONZERO)
        # This is the number of all activated elements within the ROI

        if (( $COUNTZ == $Zero )); then
            AVGZ=`echo $Zero`
            else
            AVGZ=`echo "scale=2;($SUMZ/ $COUNTZ)" | bc -l`
        fi
        
        echo "The Average of the whole surface of subject ${sub} ${cope} is $AVGZ"

# Next is to create a mask containing vertices with values >= AVGZ, let us call it posmask_avg

        echo "Creating positive mask >= $AVGZ for ${sub} ${cope}"

        wb_command -cifti-math \
            "fmri >= $AVGZ" \
            $DATA/${sub}/${cope}/${sub}.${cope}.posmask_avg.dtseries.nii \
            -var fmri $POSITIVE_NII

        POSMASK_AVG=$DATA/${sub}/${cope}/${sub}.${cope}.posmask_avg.dtseries.nii

# Next is to convert the mask with vertices >= AVGZ into the MMP parcellation, let us call it parcel.

        echo ".........Creating the MMP parcellation for ${cope} of ${sub}"
        wb_command -cifti-parcellate \
            $POSMASK_AVG \
            $HCP/${MMP_ATLAS}\
            COLUMN \
            $DATA/${sub}/${cope}/${sub}.${cope}.posmask_avg.ptseries.nii \
            -method COUNT_NONZERO
        
        PARCEL=$DATA/${sub}/${cope}/${sub}.${cope}.posmask_avg.ptseries.nii

# Next convert the parcellation file to text files

        echo "...........Converting the parcel of the ${cope} of ${sub} to a text"
        wb_command -cifti-convert \
            -to-text \
            $PARCEL \
            $DATA/${sub}/${cope}/${sub}.${cope}.ptseries.txt

        echo `paste $DATA/${sub}/${cope}/${sub}.${cope}.ptseries.txt | column -s $'\t' -t` >> $RESULTS/${cope}.txt 

    done
done

# You can convert text files to parcellations on a brain hemisphere template for the sake of visualization
# A parcellated scalar template is required. It can be found in the HCP database folders of any subject.

wb_command \
    -cifti-convert \
    -from-text \
    $RESULTS/text_to_parcel/name_of_your_text.txt \
    $HCP/????.ptseries.nii \
    # The above is a parcellation template to choose from any subject from the HCP. Choose one with one column.
    $RESULTS/parcels/name_of_your_parcel.ptseries.nii

wb_command \
    -set-map-names \
    $RESULTS/parcels/name_of_your_parcel.ptseries.nii \
    -map 1 map_name_of_preferrence
