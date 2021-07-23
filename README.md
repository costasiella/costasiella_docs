# costasiella_docs

Costasiella documentation. 

At the moment there's not much here yet, but it will be added little by little over the next year.



## Building the docs

### Install tools

python3-sphinx is a required package. For example on Fedora it can be installed like this: 

```bash
sudo dnf install python3-sphinx
```

Then we need the theme

```bash
sudo pip3 install sphinx-rtd-theme
```

Then we can build the docs by executing the following command in this directory

```bash
sphinx-build source build
```
