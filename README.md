
## Getting started in DESY-NAF

Mode info: https://confluence.desy.de/display/IS/NAF+Login%2C+WGS+and+remote+Desktop

### Login

We will use the naf-gpu login node. Here `<username>` is your DESY username.

```bash
ssh -XY <username>@naf-cms-gpu01.desy.de
```

### Create/clone repo

```bash
cd /nfs/dust/cms/user/<username>/
git clone https://github.com/shedprog/distautag-summer.git

```

### Initialization of the submodules

```bash
cd /nfs/dust/cms/user/<username>/distautag-summer
git submodule init
git submodule update
```

### Installing conda environment

```bash
cd /nfs/dust/cms/user/<username>/softDESYSUMMER/distautag-summer/DisTauMLTools
source env.sh conda
```

your anaconda environment will be installed to `DisTauMLTools/soft/conda`

to activate your environment next time you connect to naf:
```bash
source env.sh conda
```

## Understanding the input

### Very minimal discription

This project aims at optimization of the identification of the displaced taus that are daughters of long-lived stau (susy-tau) particles:

<img
  src="docs/img/stau.png"
  alt="Alt text"
  title="stau"
  style="display: inline-block; margin: 0 auto; max-width: 300px">

More information on tau particle reconstruction and identification can be found here: https://arxiv.org/pdf/1809.02816.pdf

First, when we are talking about tau we assume hydronically decaying tau (that forms jet, 
technically these jets are called AK4-jets). Tau decays to the neutral and charged pions and then they are clustered in the jet. Available tau decay modes:

<img
  src="docs/img/tau-decay.png"
  alt="Alt text"
  title="stau"
  style="display: inline-block; margin: 0 auto; max-width: 400px">

The most significant background that fakes tau-based jets are quarks/gluons/electrons/muons that create same picture in our detector and mis-reconstructed as tau. In this project we will target only discrimination versus quarks/gluons (QCD) jets. 

<img
  src="docs/img/tausignature_trans.png"
  alt="Alt text"
  title="stau"
  style="display: inline-block; margin: 0 auto; max-width: 200px">

Additionally, in our case signal jets will be displaced corresponding to the center of coordinates

<img
  src="docs/img/jet_signal_bkgr.png"
  alt="Alt text"
  title="stau"
  style="display: inline-block; margin: 0 auto; max-width: 300px">

More information about definition of signal and background classes can be found in the following presentation: https://indico.cern.ch/event/1177869/contributions/4947705/attachments/2476089/4249391/QCDvsTau_tagging_v3.pdf

### Input data study

The data with the mixed signal and background classes can be found here (to be coppied): `/nfs/dust/cms/user/mykytaua/softDeepTau/RecoML/DisTauTag/TauMLTools/FlatMerge-output-v3`

The full description of the variables can be found here: https://github.com/shedprog/DisTauMLTools/blob/33e5ab63aad7cbd8c48689a52d9d6a6e6e0817db/Analysis/interface/TauTuple.h


To open root-files locally:
```bash
root -l /nfs/dust/cms/user/mykytaua/softDeepTau/RecoML/DisTauTag/TauMLTools/FlatMerge-output-v3/ShuffleMergeFlat_0.root
>>> TBrowser l;
``` 

*Important:* In the cooresponding ntuples jetType==1 coresponds to the tau (signal), and jetType==0 to the jet

### Few useful scripts (optional):

Small, simple scripts to study the data properties can be found in the following directory:
`LLSTauTools/scripts/*`

To compare Jet Kinematic
```bash
python ./DF-JetKinemCompare.py --paths "/nfs/dust/cms/user/mykytaua/softDeepTau/RecoML/DisTauTag/TauMLTools/FlatMerge-output" "/nfs/dust/cms/user/mykytaua/softDeepTau/RecoML/DisTauTag/TauMLTools/FlatMerge-output" -na "STAU-mix" "QCD-mix" -c "jetType==1" "jetType==0" -N 1 1 -o ./output/DF-JetKinemCompare_ShuffleMergeFlat
```

To look at the kinematical properties of your signal events (here original data files are used, separated by mass of the stau)
```bash
python ./DF-MassPointsKinem.py --paths "/pnfs/desy.de/cms/tier2/store/user/myshched/ntuples-tau-pog/SUS-RunIISummer20UL18GEN-stau100_lsp1_ctau1000mm_v4/" "/pnfs/desy.de/cms/tier2/store/user/myshched/ntuples-tau-pog/SUS-RunIISummer20UL18GEN-stau250_lsp1_ctau1000mm_v4/" "/pnfs/desy.de/cms/tier2/store/user/myshched/ntuples-tau-pog/SUS-RunIISummer20UL18GEN-stau400_lsp1_ctau1000mm_v4/crab_STAU_longlived_M400/" "/pnfs/desy.de/cms/tier2/store/user/myshched/ntuples-tau-pog/SUS-RunIISummer20UL18GEN-stau100_lsp1_ctau100mm_v5/crab_STAU_longlived_M100_10cm/" "/pnfs/desy.de/cms/tier2/store/user/myshched/ntuples-tau-pog/SUS-RunIISummer20UL18GEN-stau250_lsp1_ctau100mm_v4/" "/pnfs/desy.de/cms/tier2/store/user/myshched/ntuples-tau-pog/SUS-RunIISummer20UL18GEN-stau400_lsp1_ctau1000mm_v4/crab_STAU_longlived_M400_10cm/" -na "mstau_100_1m" "mstau_250_1m" "mstau_400_1m" "mstau_100_10cm" "mstau_250_10cm" "mstau_400_10cm" -c "jet_pt>20&&abs(jet_eta)<2.4" -N 4 4 4 4 4 4 -o ./output/DF-MassPointsKinem
```

## Training the tagger ( **MAIN PART** )

### Single run
In order to have organized storage of models and its associated files, [mlflow](https://mlflow.org/docs/latest/index.html) is used from the training step onwards. At the training step, it takes care of logging necessary configuration parameters used to run the given training, plus additional artifacts, i.e. associated files like model/cfg files or output logs). Conceptually, mlflow augments the training code with additional logging of requested parameters/files whenever it is requested. Please note that hereafter mlflow notions of __run__ (a single training) and __experiment__ (a group of runs) will be used. 

The script to perform the training is [TauMLTools/Training/python/2018v1/Training_v0p1.py](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/python/2018v1/Training_v0p1.py). It is supplied with a corresponding [TauMLTools/Training/python/2018v1/train.yaml](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/python/2018v1/train.yaml) configuration file which specifies input configuration parameters. Here, [hydra](https://hydra.cc/docs/intro/) is used to compose the final configuration given the specification inside of `train.yaml` and, optionally, from a command line (see below), which is then passed as [OmegaConf](https://omegaconf.readthedocs.io/) dictionary to `main()` function. 

The specification consists of:
* the main training configuration file [`TauMLTools/Training/configs/training_v1.yaml`](https://github.com/cms-tau-pog/TauMLTools/blob/master/Training/configs/training_v1.yaml) describing `DataLoader` and model configurations, which is "imported" by hydra as a [defaults list](https://hydra.cc/docs/advanced/defaults_list/)
* mlflow parameters, specifically `path_to_mlflow` (to the folder storing mlflow runs, usually its default name is `mlruns`) and `experiment_name`.  These two parameters are needed to either create an mlflow experiment with a given `experiment_name` (if it doesn't exist) or to find the experiment to which attribute the current run
* `scaling_cfg` which specifies the path to the json file with feature scaling parameters
* `gpu_cfg` which sets the gpu related parameters 
* `log_suffix` used as a postfix in output directory names

Also note that one can conveniently [add/remove/override](https://hydra.cc/docs/advanced/override_grammar/basic/) __any__ configuration items from the command line. Otherwise, running `python Training_v0p1.py` will use default values as they are composed by hydra based on `train.yaml`.

An example of running the training would be:
```sh
python Training_v0p1.py experiment_name=run3_cnn_ho2 
```

Here, a new mlflow run will be created under the "run3_cnn_ho2" experiment (if the experiment doesn't exist, it will be created). Then, the model is composed and compiled and the training proceeds via a usual `fit()` method with `DataLoader` class instance used as a batch yielder. Several callbacks are also implemented to monitor the training process, specifically `CSVLogger`, `TimeCheckpoint` and `TensorBoard` callback. Lastly, note that the parameters related to the NN setup and initially specified in `training_v1.yaml` are overriden via the command line (`training_cfg.SetupNN.{param}=...`)