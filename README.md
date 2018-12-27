# Spacy and Textacy on AWS Lambda (Applies to other packages too)
- How to install and use spacy and textacy on aws lambda works with python 3.6

As spacy is a huge package, installing it on lambda has a lot of issues. We will use `layers` to install and use these packages. Also see https://aws.amazon.com/about-aws/whats-new/2018/11/aws-lambda-now-supports-custom-runtimes-and-layers/

Steps are:
- Create a new clean virtualenv with python3.6
- In the virtualenv, install the package using pip, ie. run `pip install spacy`
- Check all dependencies using `pip freeze` and put them in a `requirements.txt` file
- Remove dependencies that are satisfied by other layers, in my case, I have removed `numpy` and `scipy` as AWS provides these for us. The assumption is that the one provided by AWS will be maintained so one less thing to maintain and hopefully they'll optimize for size and speed of loading both? See details  
- Using a lambda docker image, install all requirements without further deps - See https://medium.com/@qtangs/creating-new-aws-lambda-layer-for-python-pandas-library-348b126e9f3e for instructions
- In the generated folder, get a list of all `__pycache__` directories and remove them. Something like `find . -type d | grep __pycache__` would give you the candidates
- Optionally remove all directories with `test` in them, however `networkx` notoriously depends on `tests`, so don't remove `tests` under `networkx`
- Remove all languages except `en` from `spacy/lang`. This is the most crucial step as it reduces the size significantly
- Creat the layer zip: `zip -r9 spacyLayer.zip .`
- Upload to s3 or directly upload to layers from the console
- Your lambda function needs to use the `scipy` layer mentioned above and your new layer you just created, both, with the `scipy` layer coming first (as spacy depends on that)
- In your lambda code, you need to install the data, e.g.`en_core_web_sm` before loading spacy, so do
```
import os
import tarfile
import urllib.request

# Check to download only once
if not os.path.isdir("/tmp/en_core_web_sm-2.0.0"):
    urllib.request.urlretrieve(
        "https://github.com/explosion/spacy-models/releases/download/en_core_web_sm-2.0.0/en_core_web_sm-2.0.0.tar.gz",
        "/tmp/en_core_web_sm-2.0.0.tar.gz",
    )
    # Extract all data
    with open("/tmp/en_core_web_sm-2.0.0.tar.gz", "rb") as f:
        t = tarfile.open(fileobj=f)
        t.extractall(path="/tmp/")
    # Cleanup
    os.remove("/tmp/en_core_web_sm-2.0.0.tar.gz")
nlp = spacy.load("/tmp/en_core_web_sm-2.0.0/en_core_web_sm/en_core_web_sm-2.0.0")

```
