Installation
Git clone this repository, and cd into directory for remaining commands

git clone https://github.com/openai/gpt-2.git && cd gpt-2

Native Installation
All steps can optionally be done in a virtual environment using tools such as virtualenv or conda.

Install tensorflow 1.12 (with GPU support, if you have a GPU and want everything to run faster)

pip3 install tensorflow==1.15
or

pip3 install tensorflow-gpu==1.15
Install other python packages:

pip3 install -r requirements.txt

Download the model data

python3 download_model.py 124M
python3 download_model.py 355M
python3 download_model.py 774M
python3 download_model.py 1558M

Running
WARNING: Samples are unfiltered and may contain offensive content.
Some of the examples below may include Unicode text characters. Set the environment variable:

export PYTHONIOENCODING=UTF-8
to override the standard stream settings in UTF-8 mode.

Unconditional sample generation
To generate unconditional samples from the small model:

python3 src/generate_unconditional_samples.py | tee /tmp/samples
There are various flags for controlling the samples:

python3 src/generate_unconditional_samples.py --top_k 40 --temperature 0.7 | tee /tmp/samples
To check flag descriptions, use:

python3 src/generate_unconditional_samples.py -- --help
Conditional sample generation
To give the model custom prompts, you can use:

python3 src/interactive_conditional_samples.py --top_k 40
To check flag descriptions, use:

python3 src/interactive_conditional_samples.py -- --help

trouble shooting

ModuleNotFoundError: No module named 'tensorflow.contrib' 
tf.contrib no longer exists in post-2.0 world.
undergrade tensorflow to 1.15 for next 3 years 
after that you will have to switch as there won't be a TF 1.16 or later

ModuleNotFoundError: No module named 'fire'
pip3 install fire

TF has no attribute Session
change tf.Session to tf.compact.v1.Session

TF has no attribute override_from_dict(json.load(f))
set HParams.parameter=value
