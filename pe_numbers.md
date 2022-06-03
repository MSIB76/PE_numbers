#!/usr/local/bin/bash

# Below is a template used to count the number of activated elements (verticies) on a brain surface. 
# Please write your desired file paths and lists of ROIs and subjects and delete the ().
# Explanation of the work of these commands and how to run a script is beyond the scope of this work.
# To understand the use of the commands and where to enter the paths to your files and output please run the command in the terminal for the explanation.
# To know more about the commands please visit the HCP workbench website https://www.humanconnectome.org/software/workbench-command or visit the HCP-Users Google group https://groups.google.com/a/humanconnectome.org/g/hcp-users
# in this template cope2 refers to the STORY paradigm and the cope4 for the STORY-MATH. 

# 1. Separate the MMP atlas into right and left label files and setup their maps

HCP=(Write here path to the parcellation folder)
MMP_ATLAS=Q1-Q6_RelatedValidation210.CorticalAreas_dil_Final_Final_Areas_Group_Colors.32k_fs_LR.dlabel.nii
wb_command -cifti-separate $HCP/${MMP_ATLAS} COLUMN -label CORTEX_LEFT (Write here path to your desired folder)/L_MMP.label.gii
wb_command -cifti-separate $HCP/${MMP_ATLAS} COLUMN -label CORTEX_RIGHT (Write here path to your desired folder)/R_MMP.label.gii
wb_command -set-map-names (Write here path to your desired folder)/L_MMP.label.gii -map 1 Left_MMP_ATLAS
wb_command -set-map-names (Write here path to your desired folder)/R_MMP.label.gii -map 1 Right_MMP_ATLAS

# 2. Convert the new dlabel.nii files into ROIs in Gifti format.

# To create the ROIs from the MMP atlas use the output files from the previous step

for MMP in (list of ROI without L or R); do
    wb_command -gifti-label-to-roi \
    (Write here path to your desired folder)/L_MMP.label.gii \
    (Write here path to your desired folder)/L_${MMP}.shape.gii \
    -map 1 -name \
    L_$MMP
    wb_command -gifti-label-to-roi \
    (Write here path to your desired folder)/R_MMP.label.gii \
    (Write here path to your desired folder)/R_${MMP}.shape.gii \
    -map 1 -name \
    R_$MMP
done

# Decide which dense to download. Here, we chose the Z-stats.

# 3. Create label dense greyordiantes that matche the brainordinate space used in a relevant template. The template we chose is the zstat1.dtseries.nii of the cope2 (STORY) found in the GrayordinatesStats/cope2.feat subfolder of the fMRI data of the same subject. You need one template only to use for all the ROI and you can choose any beta map file.

for sub in (List of subjects); do
    for MMP in (list of ROI without L or R); do
        COPE2=(Write here path to a single cope2.feat folder from any subject from the HCP data base)
        wb_command -cifti-create-dense-from-template \
            $COPE2/zstat1.dtseries.nii \
            (Write here path to your desired folder)/L.${MMP}.dscalar.nii \
            -metric \
            CORTEX_LEFT \
            (Write here path to your desired folder)/L_${MMP}.shape.gii
        wb_command -cifti-create-dense-from-template \
            $COPE2/zstat1.dtseries.nii \
            (Write here path to your desired folder)/R.${MMP}.dscalar.nii \
            -metric \
            CORTEX_RIGHT \
            (Write here path to your desired folder)/R_${MMP}.shape.gii
    done
done

# 4. Separate the dense in the ROI
## This step might not be necessary for you. However, it is good for visualization and merging of dscalar files later, it is just our preference. You can skip it and do the measurement using the whole Cifti time series (dtseries) files.

for sub in (Write here List of subjects); do
    COPE2=(Write here path to ${sub} specific cope2.feat folder containing the zstat1.dtseries.nii file) 
    COPE4=(Write here path to ${sub} specific cope4.feat folder containing the zstat1.dtseries.nii file) 
    for h in L R; do
        for MMP in (Write here list of ROI without L or R); do
            echo "..... Create Z-stats cut of STORY for ${sub} ${h} ${MMP} "
            wb_command -cifti-math \
            "fmri * label" \
            (Write here path to your desired folder)/${sub}.${h}.${MMP}.cope2_cut.dtseries.nii \
            -var fmri $COPE2/zstat1.dtseries.nii \
            -var label (Write here path to your desired folder)/${h}.${MMP}.dscalar.nii
            echo "..... Create Z-stats cut of STORY-MATH for ${sub} ${h} ${MMP} "
            wb_command -cifti-math \
            "fmri * label" \
            (Write here path to your desired folder)/${sub}.${h}.${MMP}.cope4_cut.dtseries.nii \
            -var fmri $COPE4/zstat1.dtseries.nii \
            -var label (Write here path to your desired folder)/${h}.${MMP}.dscalar.nii
        done
    done
done

# 5. Thresholding the activated elements at a value higher than the average value.
## Our approach to measure the number of elements activated (Here are vertices) is by counting the number of elements that exceeded the average threshold within the same ROI. 
## Accordingly, we need first, to measure the average threshold, followed by counting the number of activated elements.

Zero=0
for sub in (Write here List of subjects); do
    for MMP in (Write here list of ROI without L or R); do
        for cope in cope2 cope4; do
            for h in L R; do
                wb_command -cifti-math \
                "fmri > 0" \
                (Write here path to your desired folder)/${sub}.${h}.${MMP}.${cope}.posmask.dtseries.nii \
                -var fmri (Write here path to your desired folder)/${sub}.${h}.${MMP}.${cope}_cut.dtseries.nii
                # This create a Cifti file with elements with positive BOLD values only (POSMASK)
                POSMASK=(Write here path to your desired folder)/${sub}.${h}.${MMP}.${cope}.posmask.dtseries.nii
                wb_command -cifti-math \
                "fmri * pos" \
                (Write here path to your desired folder)/${sub}.${h}.${MMP}.${cope}.pos.dtseries.nii \
                -var fmri (Write here path to your desired folder)/${sub}.${h}.${MMP}.${cope}_cut.dtseries.nii \
                -var pos $POSMASK
                # This creates a Cifti time series (dtseries) files containining the positive BOLD within the specified regions, ready to be measured in the next step.
                POS=(Write here path to your desired folder)/${sub}.${h}.${MMP}.${cope}.pos.dtseries.nii
                SUM=$(wb_command -cifti-stats $POS -reduce SUM)
                # This is the summation value of the activated elements z-stat within the ROI
                COUNT=$(wb_command -cifti-stats $POS -reduce COUNT_NONZERO)
                # This is the number of all activated elements witihin the ROI
                # Now to measure the mean threshold as follows:
                if (( $COUNT == $Zero )); then
                    AVG=`echo $Zero`
                    else
                    AVG=`echo "scale=2;($SUM/ $COUNT)" | bc -l`
                fi
                echo "The Average of the ${sub}.${h}.${MMP}.${cope} is $AVG"
                # Now to use that mean values (AVG) to threshold the dtseries files:
                if [ "$AVG" == "$Zero" ]; then
                    echo "Abovemean for ${sub}.${h}.${MMP}.${cope} Stopped"
                    else
                    wb_command -cifti-math "fmri > $AVG" (Write here path to your desired folder)/${sub}.${h}.${MMP}.${cope}.abovemean.dtseries.nii -var fmri $POS
                fi
            done
        done
    done
done

# 6. Export the number of activated elements into data sheet
Zero=0
for sub in (Write here List of subjects); do
    for MMP in (Write here list of ROI without L or R); do
        for cope in cope2 cope4; do
            # This will create masks of the activated elements above the mean threshold
            # Now it time to export these values to text files.
            POSROIL=(Write here path to your desired folder)/${sub}.L.${MMP}.${cope}.abovemean.dtseries.nii
            POSROIR=(Write here path to your desired folder)/${sub}.R.${MMP}.${cope}.abovemean.dtseries.nii
            if [ ! -f $POSROIL ]; then
                LEFT=`echo $Zero`
                else
                LEFT=$(wb_command -cifti-stats $POSROIL -reduce SUM)
            fi
            if [ ! -f $POSROIR ]; then
                RIGHT=`echo $Zero`
                else
                RIGHT=$(wb_command -cifti-stats $POSROIR -reduce SUM)
            fi
            echo "$LEFT" >> (Write here path to your desired folder)/L.${MMP}.${cope}.csv
            echo "$RIGHT" >> (Write here path to your desired folder)/R.${MMP}.${cope}.csv
            echo `paste (Write here path to your desired folder)/L.${MMP}.${cope}.csv | column -s $'\t' -t` >> (Write here path to your desired folder)/L.${cope}.csv 
            echo `paste (Write here path to your desired folder)/R.${MMP}.${cope}.csv | column -s $'\t' -t` >> (Write here path to your desired folder)/R.${cope}.csv
        done
    done
done

# After finishing statistical analysis and generating the effect size value for each ROI we create text files and plot these results on a brain hemisphere template for the sake of visualization
# A parcellated scalar template is required. Those can be found in the feat folder.

wb_command -cifti-convert (Write here path to text file).txt (Write here path to any subject template file).pscalar.nii (Write here path to output file of parcellated values of effect size cope2).pscalar.nii
wb_command -set-map-names (Write here path to output file of parcellated values of effect size cope2).pscalar.nii -map 1 (Write here name of the map)
# Do the same for the cope4
