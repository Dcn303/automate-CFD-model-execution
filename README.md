# automate-CFD-model-execution
##script for Running CFD model 3D lid driven cavity 8M on HPC cluster using RapidCFD CFD simulation tools 
#! /bin/bash
mkdir gpu_runs
cd gpu_runs
test_case=$1
start=0
end=4
p_maxIter=650
device_flag_for_2_gpus="-devices \"(0 1)\""
device_flag_for_4_gpus="-devices \"(0 1 0 1)\""
device_flag_for_6_gpus="-devices \"(0 1 0 1 0 1)\""
device_flag_for_8_gpus="-devices \"(0 1 0 1 0 1 0 1)\""
for(( i=${start};i<=${end};i++ ))
do

#COPY THE GPU SCRIPT
 gres_value=2
        total_node=${i}
        total_no_of_gpus_or_core=$(( ${total_node}*2 ))
if [ ${total_no_of_gpus_or_core} -eq 0 ] ; then
        #COPYING WHEN EXECUTING ON SINGLE GPU
        cp -rf ../${test_case} ${test_case}_gpu_1
        cd ${test_case}_gpu_1
else
#COPYING THE TEST CASES
        cp -rf ../${test_case} ${test_case}_gpu_${total_no_of_gpus_or_core}
#ENTERING THE TEST CASES
        cd ${test_case}_gpu_${total_no_of_gpus_or_core}
fi

        if [ ${total_no_of_gpus_or_core} -eq 0 ] ; then
                gres_value=1
cat <<EOF > gpu.sh
#! /bin/bash
#SBATCH -J 3D_lid_driven_cavity
#SBATCH -p gpu
#SBATCH -N 1
#SBATCH --gres=gpu:${gres_value}
#SBATCH --ntasks-per-node=1
(time icoFoam ) > log.icoFoam 2>&1
EOF
else
        device_flag_temp="device_flag_for_${total_no_of_gpus_or_core}_gpus"
        device_flag=$( eval echo "\$$device_flag_temp" )
cat <<EOF >gpu.sh
#! /bin/bash
#SBATCH -J 3D_lid_driven_cavity
#SBATCH -p gpu
#SBATCH -N ${total_node}
#SBATCH --gres=gpu:${gres_value}
#SBATCH --ntasks-per-node=2
N=$(( ${i}*2 ))
(time mpirun -np \$N icoFoam -parallel ${device_flag} > log.icoFoam 2>&1
EOF
        fi
#END OF COPYING THE GPU SCRIPT

        total_no_of_core=${total_no_of_gpus_or_core}
        no_of_subdomain=${total_no_of_gpus_or_core} #SETTING THE NUMBER OF DECOMPOSITION VALUE
        if [ ${total_no_of_core} -eq 0 ] ; then
                total_no_of_core=1
                no_of_subdomain=1;
        fi

if [ ${total_no_of_core} -lt 32 ] ; then
        decomposePar="(time decomposePar) > log.decomposePar 2>&1"
        #IF IT IS RUNNING FOR THE SINGLE GPU OR CPU THEN WE DONT DO DECOMPOSEPAR
        if [ ${total_no_of_core} -eq 0 ] ; then
                        decomposePar=" "
        fi
cat <<EOF >cpu.sh
#! /bin/bash
#SBATCH -J 3D lid driven cavity
#SBATCH -p cpu
#SBATCH -N 1
#SBATCH --ntasks-per-node=${total_no_of_core}
N=${total_no_of_core}
(time blockMesh) > log.blockMesh 2>&1
${decomposePar}
EOF
fi

#ENTERING THE SYSTEM FOLDER
cd system
rm -rf fvSchemes
cat<<EOF >fvSchemes
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  v2006                                 |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "system";
    object      fvSchemes;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

ddtSchemes
{
    default         Euler;
}

gradSchemes
{
    default         Gauss linear;
    grad(p)         Gauss linear;
}
//the main difference between the zeptoFOAM fvSchemes and the fvSchemes of RapidCFD is two line only
//i.e. in divSchemes //check the comment made in the divSchemes
divSchemes
{
//    default         none;
    default         Gauss linear;   //earlier this line was:=  default none; //the value is set according to zeptoFOAM
    div(phi,U)      Gauss linear;
    div((nuEff*dev2(T(grad(U))))) Gauss linear; //this line is added accordingly as zeptoFOAM
}

laplacianSchemes
{
    default         Gauss linear orthogonal;
}

interpolationSchemes
{
    default         linear;
}

snGradSchemes
{
    default         orthogonal;
}

fluxRequired
{
    default         no;
    p               ;
}

// ************************************************************************* //
EOF
rm -rf fvSolution
cat <<EOF >fvSolution
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  2.3.0                                 |
|   \\  /    A nd           | Web:      www.OpenFOAM.org                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "system";
    object      fvSolution;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

solvers
{
    p
    {
       solver          PCG; //this was earlier settingsin RapidCFD 3D_lid driven M
//        preconditioner  AINV; //this was earlier setting in RapidCFD 3D_lid driven M
        preconditioner  diagonal; //this is set according to zeptoFOAM
//        tolerance       0;             //this was earlier settings in RapidCFD 3D_lid driven M
//        relTol          0;             //this was earlier settings in RapidCFD 3D_lid driven M
        tolerance       1e-06;           //this is set according to zeptoFOAM
        relTol          0.05;            //this is set according to zeptoFOAM
        maxIter         ${p_maxIter};
    }
   pFinal
    {
        \$p;
        relTol          0;
    }

    U
    {
//        solver          smoothSolver;  //this was earlier settings in RapidCFD 3D_lid driven M
        solver          PBiCG;          //this is set according to zeptoFOAM
//        smoother        GaussSeidel; // this is not set in zeptoFOAM
        preconditioner          diagonal; // this is set according to zeptoFOAM
//        tolerance       0;           //this was earlier settings in RapidCFD 3D_lid driven M
//        relTol          0;           //This was earlier settings in RapidCFD 3D_lid driven M
//        maxIter         5;           //this was earlier settings in RapidCFD 3D_lid driven M
        tolerance       1e-10;         //This is set according to zeptoFOAM
        relTol          0;              //This is set according to zeptoFOAM
        maxIter         10;             //This is set according to zeptoFOAM
    }
}

PISO
{
    nCorrectors     2;
    nNonOrthogonalCorrectors 0;
    pRefCell        0;
    pRefValue       0;
}


// ************************************************************************* //
EOF

rm -rf decomposeParDict
cat <<EOF > decomposeParDict
  /*--------------------------------*- C++ -*----------------------------------*\
  | =========                 |                                                 |
  | \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
  |  \\    /   O peration     | Version:  v2006                                 |
  |   \\  /    A nd           | Website:  www.openfoam.com                      |
  |    \\/     M anipulation  |                                                 |
  \*---------------------------------------------------------------------------*/
  FoamFile
  {
      version     2.0;
      format      ascii;
      class       dictionary;
      location    "system";
      object      decomposeParDict;
  }
  // * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

  numberOfSubdomains ${no_of_subdomain};

  method          scotch;


  // ************************************************************************* //
EOF
#comming out of the system folder
cd ..

#ENTERING THE 0 FOLDER
cd 0
rm -rf U
cat << EOF > U
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  v2006                                 |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       volVectorField;
    object      U;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

dimensions      [0 1 -1 0 0 0 0];

internalField   uniform (0 0 0);

boundaryField
{
    movingWall
    {
        type            fixedValue;
        value           uniform (1 0 0);
    }

    fixedWalls
    {
//        type            fixedValue;           //This was added earlier in RapidCFD
//        value           uniform (0 0 0);      //This was added earlier in RapidCFD
        type            noSlip;                 //This is added accordingly as zeptoFOAM
    }

    frontAndBack
    {
        type            empty;
    }
}
EOF
#comming out of the 0 folder
cd ..
#ENTERING CONSTANT FOLDER
cd constant
rm -rf transportProperties
cat <<EOF > transportProperties
/*--------------------------------*- C++ -*----------------------------------*\
| =========                 |                                                 |
| \\      /  F ield         | OpenFOAM: The Open Source CFD Toolbox           |
|  \\    /   O peration     | Version:  v2006                                 |
|   \\  /    A nd           | Website:  www.openfoam.com                      |
|    \\/     M anipulation  |                                                 |
\*---------------------------------------------------------------------------*/
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    location    "constant";
    object      transportProperties;
}
// * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * //

nu              nu [0 2 -1 0 0 0 0] 0.01;

// ************************************************************************* //
EOF
#EXITING CONSTANT FOLDER
cd ..
#comming out of the case
cd ..
done

