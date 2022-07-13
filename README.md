# unofficial_irods_documentation
Best Practices and System Administrator documentation for iRODS systems by the community. NOT affiliated with RENCI or the iRODS project.

## Background

Proposed at lighting talk by John Constable at the [2022 iRODS User Group Meeting](https://irods.org/ugm2022/).

## Contributing

### Technical Notes

Set up your development environment with

```
python3 -m venv unofficial_irods_documentation
source unofficial_irods_documentation/bin/activate
pip install -r requirements.txt
pre-commit install
# gramma command line tool https://caderek.github.io/gramma/
mkdir -p ~/bin
cd ~/bin
wget https://github.com/caderek/gramma/releases/download/v1.5.0/gramma-linux64-v1.5.0.zip
unzip gramma-linux64-v1.5.0.zip
rm gramma-linux64-v1.5.0.zip
```

### Getting started

[Getting started docs from mkdocs](https://www.mkdocs.org/getting-started/)

### checks

Please check content with `gramma` before raising an MR.
	* `gramma check -m -l "en-GB" unofficial_irods_documentation/docs/index.md`


### Code Of Conduct

Inspired by the [irods.org code of conduct](https://irods.org/code-of-conduct/).

This repository is dedicated to providing a safe and enjoyable experience for everyone regardless of gender, gender identity, gender expression, sexual orientation, disability, physical appearance, body size, race, religion, national origin, operating system or text editor of choice. We do not tolerate harassment of contributors in any form.

If a contributor, including but not limited to those writing content, raising and commenting on issues, engages in harassing behaviour, the maintainers of the repository may take any action they deem appropriate, including warning the offender or expelling the offender from the repository.

If you are being harassed, notice that someone else is being harassed, or have any other concerns, please contact a maintainer.

The authors of this repository intend for these policies to meet the needs of our constituents to have a positive documentation experience, and we welcome comments and suggestions from the community. Please contact raise an GitHub issue if you would like to provide feedback.

# Yaks Shaved

2
