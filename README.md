# Smart Grid Store - 4.15.x

Smart Grid Store is a distribution of the Berkeley Tree Database \(BTrDB\) that packages  
BTrDB along with tools for working with smart grid data in containers for easy deployment  
on a Kubernetes cluster. This guide is for version 4.15.x

Smart Grid Store, or SGS, consists of the following components:

* BTrDB: the timeseries database that stores all the data
* Multi Resolution Plotter \(Mr. Plotter\): an efficient, uncomplicated plotter for visualizing and exploring massive datasets in real-time
* AdminCLI: A supervisory admin text-based console that allows monitoring and configuring the cluster
* pmu2btrdb: A high performance direct-path ingress daemon for Power Standards Lab microsynchrophasors
* receiver/sync2q: A buffered-path ingress daemon pair for uPMUs that preserves the raw files produced by the devices for debugging or recovery at the expense of performance
* c37ingress: An ingress daemon for IEEE C37.118 synchrophasor data
* gepingress: An ingress daemon for GEP

We are working on developing the following components for the latest version of SGS:

* DISTIL: continuous analysis and transformation of BTrDB streams
* Jupyter notebooks: a preconfigured python notebook setup for quick custom analysis

SmartGridStore relies upon third party software:

* [Kubernetes](https://kubernetes.io/): a container orchestration framework
* [Ceph](https://ceph.com/): a software defined storage layer

Both of these dependencies are rock solid, and have available enterprise support contracts. Note that at present we do not support [OpenSHIFT](https://www.openshift.com/), the commercial Kubernetes distribution by Red Hat, as we use features in Kubernetes upstream that have not made it to openshift yet, but if there is enough interest we may reconsider this choice.

An enterprise support contract for Smart Grid Store is available upon request, and is provided by [PingThings Inc.](http://www.pingthings.io/).

