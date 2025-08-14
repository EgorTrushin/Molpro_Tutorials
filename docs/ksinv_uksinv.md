## KSINV and UKSINV programs
![Status](https://img.shields.io/static/v1.svg?label=Status&message=Preliminary%20Version&color=orange)

The programs `KSINV` and `UKSINV` solve the inverse Kohn-Sham (KS) problem in the spin-restricted and spin-unrestricted cases, respectively.
In the spin-restricted case, the KS inversion determines the KS potential for a given target density.
In the spin-unrestricted case, the KS potentials for the $\alpha$ and $\beta$ spin channels are determined for given target electron densities for $\alpha$ and $\beta$ electrons.  
The program is designed to work with densities provided by many-body methods such as Coupled Cluster (CC) or Configuration Interaction (CI). The implementation of KS inversion follows the so-called response function approach proposed in Ref. 1. The actual implementation for closed-shell systems was described and tested in Ref. 2. The extension to open-shell systems was described in Ref. 3.

---

**NOTE:** We have tutorial which provides practical Hands-on examples about the use of `KSINV` and `UKSINV` programs and post-processing of results of calculations. This tutorial is a good supplement to this documentation. [Link to Tutorial](https://github.com/EgorTrushin/tutorials/blob/main/KS_inversion.ipynb)

---

**Important:** Read carefully the information below about the selection of basis sets and the `thr_fai_oep` parameter:  
Kohn-Sham inversion calculations requires to specify two basis sets, namely orbital and OEP basis sets. The orbital basis set is the primary basis set for e.g. the representation of orbitals and the calculation of energy contributions. The OEP basis set is the auxiliary basis set for representing e.g. the exchange-correlation potential. 
Gaussian basis set KS inversion methods for a long time suffered from numerical instabilities resulting from problems in balancing the orbital basis sets with the auxiliary basis sets.
Present implementation of KS inversion in Molpro use a preprocessing of the auxiliary basis set that enables such balancing in an automatic fashion for standard Gaussian basis sets.
The preprocessing step removes linear combinations of auxiliary basis functions that couple poorly to products of occupied times unoccupied Kohn-Sham orbitals. It involves the threshold `thr_fai_oep`, which determines how many linear combinations of auxiliary basis functions are removed and varies with respect to the size of the orbital basis set used. This scheme was tested using Dunning correlation consistent basis sets aug-cc-pwCVXZ (X = T, Q, 5) and recommended thresholds are

| Orbital basis set | **thr_fai_oep** |
| :----: | :----: |
| aug-cc-pwCVTZ  |  5e-2   |
| aug-cc-pwCVQZ  | 1.7e-2  |
| aug-cc-pwCV5Z  |  5e-3   |

These thresholds are also expected to work for orbital basis sets without augmentation cc-pwCVXZ (X = T, Q, 5) and without core-polarization functions aug-cc-pVXZ (X = T, Q, 5).
 
As an auxiliary basis set (OEP), the aug-cc-pVXZ/mp2fit (X=T,Q,5) family of basis sets is recommended. In particular, it is recommended to use the aug-cc-pVDZ/mp2fit auxiliary basis sets for atoms up to neon and the aug-cc-pVTZ/mp2fit auxiliary basis sets for heavier atoms.
 
Def2 basis sets of Ahlrichs family of similar quality also might work, but were not tested extensively in practice.

---

Example input file for KS inversion of the reference density provided by the CCSD method for the CO molecule:
```
basis={
default,aug-cc-pwCVQZ ! orbital basis
set,oep;default,aug-cc-pVDZ/mp2fit ! ONote that we have to explicitly specify the number of and electrons, see noa and nob parameters.EP basis
}

symmetry,nosym ! KSINV does not use symmetry

angstrom
geometry={
2

C        0.000000    0.000000   -0.646514
O        0.000000    0.000000    0.484886
}

hf,maxit=30 ! HF calculations

{CCSD;core;dm,1325.1} ! reference CCSD calculation
e_ref_ccsd=energy ! save energy to provide as a reference to KSINV

! save relaxed density to the first record - required by KSINV
{matrop;load,den,den,1325.1;save,den,1325.1}

acfd;ksinv,refden=1325.1,e_ref=e_ref_ccsd,thr_fai_oep=1.7d-2 ! KSINV calculation
```
It may be useful to split the reference density and KS inversion calculations into two separate calculations. In this case, the reference density can be saved as:
```
basis={
default,aug-cc-pwCVQZ ! orbital basis
}

symmetry,nosym ! KSINV does not use symmetry

angstrom
geometry={
2

C        0.000000    0.000000   -0.646514
O        0.000000    0.000000    0.484886
}

hf,maxit=30 ! HF calculations

{CCSD;core;dm,1325.1} ! reference CCSD calculation
{matrop;load,den,den,1325.1;write,den,dm.dat,status=rewind,prec=sci}
```
And then the KS inversion can be performed:
```
basis={
default,aug-cc-pwCVQZ ! orbital basis
set,oep;default,aug-cc-pVDZ/mp2fit ! OEP basis
}

symmetry,nosym ! KSINV does not use symmetry

angstrom
geometry={
2

C        0.000000    0.000000   -0.646514
O        0.000000    0.000000    0.484886
}

hf,maxit=0 ! HF calculation with 0 iterations, KSINV uses this for initialization

{matrop;read,den,type=density,file=dm.dat;save,den,1325.1} ! read stored density matrix

acfd;ksinv,refden=1325.1,e_ref=-113.285493180105,thr_fai_oep=1.7d-2 ! KSINV calculation
```

Example input file for spin-unrestricted KS inversion for open-shell system, the Li atom, with the reference density provided by FCI:
```
basis={
default,aug-cc-pwCVTZ
set,oep
default,aug-cc-pVDZ/mp2fit
}

symmetry,nosym
angstrom
geometry={
1

Li     0.0000000    0.0000000    0.00000000
}


hf 
{fci;dm,3000.2;core}
e_ref=energy

acfd;uksinv,maxit=100,refden=3000.2,e_ref=e_ref,thr_fai_oep=5d-2
```

Example input file for spin-restricted KS inversion for open-shell system, the Li atom, with the reference density provided by FCI:
```
basis={
default,aug-cc-pwCVTZ
set,oep
default,aug-cc-pVDZ/mp2fit
}

symmetry,nosym
angstrom
geometry={
1

Li     0.0000000    0.0000000    0.00000000
}


hf 
{fci;dm,3000.2;core}
e_ref=energy

acfd;ksinv,maxit=100,refden=3000.2,e_ref=e_ref,vhoep=1,thr_fai_oep=5d-2,noa=2,nob=1
```
Note that we have to explicitly specify the number of $\alpha$ and $\beta$ electrons, see `noa` and `nob` parameters.

The following options are available for the `KSINV` and `UKSINV` programs:
- **refden** record from which to read the reference density
- **orb** record from which the occupation numbers are read (default: ‘2100.2’)
- **save** record in which the resulting orbital coefficients, eigenvalues, etc. are written (default: '2101.2')
- **e_ref** total energy of calculations from which reference density is provided
- **dfit** if set to $\neq$ 0, enable density fitting for two-electron integrals (default: ’0’)  
- **thr_inversion** convergence threshold for self-consistent procedure (default '1d-10')
- **maxit** maximum number of iterations (default '30')
- **minit** minimum number of iterations (default '3')
- **mixing_a**  real parameter to control the update of the exchange correlation potential in the KS inversion procedure. The parameter corresponds to $a$ in Appendix A of Ref. [1] (default: '0.1')
- **mix_switch_iter** integer parameter to control the update of the exchange correlation potential in the KS inversion procedure. The parameter corresponds to $i$ in Appendix A of Ref. [2] (default: '3')
- **backfilter** if set to $\neq$ 0, Eq. (52) of Ref. [1] is applied at each iteration to remove contributions not contained in the space spanned by the current iteration's auxiliary basis set (default: '1')
- **thr_sym** threshold for symmetrization of the OEP basis set to enforce that OEP basis exhibits full symmetry of molecule. Set the threshold to 1d-10 to enable symmetrization. (default: ‘0d0’)
- **thr_fai_oep** threshold for processing OEP basis according to Section IIB5 in Ref. [3] (default: ‘1.7d-2’)
- **thr_overlap_oep** threshold for processing OEP basis according to Section IIB2 in Ref. [3] (default: ‘1d-99’)
- **vref_fa** if set to $\neq$ 0, enable the use of the Fermi-Amaldi potential as reference potential when applying the charge condition. Otherwise, the reference potential is constructed according to Eq. (45) of Ref. [4] (default: '1')
- **thr_oep** threshold for throwing out contributions corresponding to small eigenvalue differences appearing in the denominator when constructing the so-called lambda term $1/(\varepsilon_a - \varepsilon_i)$ of the static Kohn-Sham response function (default: ‘1d-6’)
- **solve** matrix inversion methods to solve the OEP equation. The different options are: GESV, TSVD, GTSVD. GESV corresponds to a direct solution without any regularization technique. TSVD and GTSVD correspond to two solutions with regularization according to Eqs. (55) and (56) of Ref. [5], respectively. (default: 'GTSVD')
- **thr_solve** threshold used during matrix inversion to solve the OEP equation with TSVD and GTSVD methods. Note that the default threshold of 1d-99 results in the absence of regularization (default: ‘1d-99’)
- **plot_always** if set to $\neq$ 0, enable writing of data-files for plotting for every iteration. Otherwise, only final results are written. (default: '0')
- **plot_vc** if set to $\neq$ 0, enable writing of file with correlation potential (default: '0')
- **plot_vx** if set to $\neq$ 0, enable writing of file with exchange potential (default: '0')
- **plot_vxc** if set to $\neq$ 0,enable writing of file with exchange-correlation potential (default: '0')
- **plot_vref** if set to $\neq$ 0, enable writing of file with reference potential (default: '0')
- **plot_rho_ks** if set to $\neq$ 0, enable writing of file with Kohn-Sham density (default: '0')
- **plot_rho_ref** if set to $\neq$ 0, enable writing of file with reference reference (default: '0')
- **plot_rho_diff** if set to $\neq$ 0, enable writing of file with difference between Kohn-Sham and reference density (default: '0')
- **plot_x** if set to $\neq$ 0, enable writing of file with plotting data along x-axis (default: '0')
- **plot_y** if set to $\neq$ 0, enable writing of file with plotting data along y-axis (default: '0')
- **plot_z** if set to $\neq$ 0, enable writing of file with plotting data along z-axis (default: '0')
- **plot_xy** if set to $\neq$ 0, enable writing of file with plotting data for xy-plane (default: '0')
- **plot_yz** if set to $\neq$ 0, enable writing of file with plotting data for yz-plane (default: '0')
- **plot_xz** if set to $\neq$ 0, enable writing of file with plotting data for xz-plane (default: '0')
- **gridsize** determine the number of grid points for stored data (default: '2048')
- **plotrange** determine the range for which plotting data will be evaluated as [-plotrange:plotrange] (default: '20d0')
- **verb** determines the level of verbosity in the output file, integer values of 0, 1, 2, and 3 provide different levels of verbosity (default ’0’)
- **noa** number of electron in $\alpha$ spin channel, required for calculations of open-shell systems with `KSINV`.
- **nob** number of electron in $\beta$ spin channel, required for calculations of open-shell systems with `KSINV`.
- **vhoep** if set to $\neq$ 0, enable the calculation of the Hartree potential from the representation in the OEP basis instead of the construction from the density matrix as in the Hartree-Fock calculation (default: ‘0’)
- **space_sym** if set to $\neq$ 0, enable the space-symmetrization. When active, sets vhoep=1 thr_sym=1d-10. (default: '0')
- **vref_fa_sameab** if set to $\neq$ 0, force the Fermi-Amaldi reference potential to be the same for $\alpha$ and $\beta$ spin channels in `UKSINV` calculations (default: ‘0’)
- **homo** if set to $\neq$ 0, enable the use of the HOMO condition. Note that epsilon_major/epsilon_minor must be specified to get meaningful results. (default '0')
- **epsilon_major** when homo=1, specifies the energy of HOMO orbital in $\alpha$ spin channel.
- **epsilon_minor** when homo=1, specifies the energy of HOMO orbital in $\beta$ spin channel.
- **density_test** if set to $\neq$ 0, enable density difference test. $|\rho_{ref}(r) - \rho_{KS}(r)|$ is evaluated on real space grid and integrated. (default '0')
- **thr_den_intgr** threshold for density difference integration test (default '1d-6')

Since KS exchange, correlation and exchange-correlation potentials are important outcomes of KS inversion, we provide an illustration of how to plot these potentials. Let us assume that we have performed calculations for CO with the following options:
```
acfd;ksinv,refden=1325.1,e_ref=-113.285493180105,thr_fai_oep=1.7d-2,\
plot_vx=1,plot_vc=1,plot_vxc=1,plot_vref=1,plot_z=1
```
At the end the files `vref-total-final-z.csv`, `vx-total-final-z.csv`, `vc-total-final-z.csv` and `vxc-total-final-z.csv` are written which contain required data for plotting potentials. 
The exchange-correlation potential consist of a reference exchange potential $v_x^{ref}$ and a remainder $v_{\bar{x}c}$:
$$v_{xc}=v_x^{ref}+v_{\bar{x}c}$$
We also have access to the individual part of the reminder $v_{\bar{x}c}$, namely a reminder of the exchange potential $v_{\bar{x}}$ and the correlation potential $v_c$.
The correlation potential $v_c$ can be plotted as it is. The exchange potential is the sum of the reference exchange potential $v_x^{ref}$ and the corresponding reminder $v_{\bar{x}}$:
$$v_x=v_x^{ref}+v_{\bar{x}}$$


```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

vref = pd.read_csv('vref-total-final-z.csv')
vx = pd.read_csv('vx-total-final-z.csv')
vc = pd.read_csv('vc-total-final-z.csv')
vxc = pd.read_csv('vxc-total-final-z.csv')

plt.plot(vx.z, vref.value+vx.value, color='orangered', label='$v_x$')
plt.plot(vc.z, vc.value, color='orange', label='$v_c$')
plt.plot(vxc.z, vref.value+vxc.value, color='dodgerblue', label='$v_{xc}$')

plt.ylabel('Potential (Hartree)', fontsize=16)
plt.xlabel('r (Bohr)', fontsize=16)
plt.ylim(-2, 0.35)
plt.xlim(-5, 5)
plt.legend(frameon=False, fontsize=14)
plt.show()
```
![](ksinv_co.png)

We have tested several post-Hartree-Fock methods for preparing reference densities. Below are examples of how to perform calculations using these methods, which store the density matrix in a record and in a file: 

For FCI in closed-shell case:
```
{FCI;core;dm,1325.1}
{matrop;load,den,den,1325.1;write,den,dm.dat,status=rewind,prec=sci}
```

For CCSD:
```
{CCSD;core;dm,1325.1}
{matrop;load,den,den,1325.1;write,den,dm.dat,status=rewind,prec=sci}
```

For CCSD(T):
```
{CCSD(T);core;dm,1325.1}
{matrop;load,den,den,1325.1;write,den,dm.dat,status=rewind,prec=sci}
```

Then one can read `dm.dat` from disc as `{matrop;read,den,type=density,file=dm.dat;save,den,1325.1}`.

For FCI in open-shell case:
```
{fci;dm,1325.1;core}
{matrop;export,1325.1,dm.dat,status=rewind,prec=sci}
```

For RS2:
```
{rs2;dm,1325.1;core}
{matrop;export,1325.1,dm.dat,status=rewind,prec=sci}
```

For CISD
```
{ci;save,density=1325.1,spinden;core}
{matrop;export,1325.1,dm.dat,status=rewind,prec=sci}
```

For AQCC:
```
{aqcc;save,density=1325.1,spinden;core}
{matrop;export,1325.1,dm.dat,status=rewind,prec=sci}
```

Then one can read `dm.dat` from disc as: `{matrop;import,2140.2,file=dm.dat}`

CCSD and CCSD(T) implementation in Molpro does not allow to store density matrix when applied to open-shell systems. Therefore, we must use CI code in this case, namely FCI, RS2, CISD, and AQCC. RS2 can store only total density, thus it can be used only in the spin-restricted inversion. For RS2, CISD, and AQCC, one can also use multi-reference variants which potentially are more accurate.

When energy contributions are of interest, it is essential to evaluate the error measures of the KS inversion because large errors render energy contributions meaningless. These measures are printed in the section of the output file labeled "Error Measures" and include:
- Hartree energy error
- External energy error
- Lieb error

You can read about these error measures in Ref. 6. Strategies for reducing these error measures are discussed in Ref. 6 and in the tutorial. The `density_test` option can also be used to evaluate integrated density error.

Density fitting can be used in KS inversion as follows: Let's first perform the calculation to store the density matrix.
```
basis={
default,aug-cc-pwCVTZ ! orbital basis
}

symmetry,nosym ! KSINV does not use symmetry

angstrom
geometry={
2

C        0.000000    0.000000   -0.646514
O        0.000000    0.000000    0.484886
}

hf,maxit=30 ! HF calculations

{CCSD;core;dm,1325.1} ! reference CCSD calculation
{matrop;load,den,den,1325.1;write,den,dm.dat,status=rewind,prec=sci}
```
Then, we can perform the KS inversion with `gdirect` option and density fitting.
```
gdirect

basis={
default,aug-cc-pwCVTZ ! orbital basis
set,oep;default,aug-cc-pVDZ/mp2fit ! OEP basis
set,dfit;default,aug-cc-pwCV5Z/mp2fit ! density fitting basis
}

symmetry,nosym ! KSINV does not use symmetry

angstrom
geometry={
2

C        0.000000    0.000000   -0.646514
O        0.000000    0.000000    0.484886
}

{matrop;read,den,type=density,file=dm.dat;save,den,1325.1} ! read stored density matrix

df-hf,maxit=0,df_basis=dfit ! HF calculation with 0 iterations
{cfit,basis_coul=dfit,basis_exch=dfit}

acfd;ksinv,refden=1325.1,e_ref=-113.251028274562,thr_fai_oep=5d-2,dfit=1
```
It is necessary to read and store the density matrix in a record first and only after this to run Hartree-Fock and KS inversion calculations.

**Bibilography:**  
[1] A. Görling, [Phys. Rev. A](https://journals.aps.org/pra/abstract/10.1103/PhysRevA.46.3753) 46, 3753 (1992)  
[2] J. Erhard, E. Trushin, A. Görling [J. Chem. Phys.](https://aip.scitation.org/doi/full/10.1063/5.0087356) 156, 204124 (2022)  
[3] J. Erhard, E. Trushin, A. Görling [J. Chem. Phys.](https://doi.org/10.1063/5.0239422) 162, 034116 (2025)  
[4] E. Trushin, A. Görling, [J. Chem. Phys.](https://aip.scitation.org/doi/full/10.1063/5.0056431) 155, 054109 (2021)  
[5] P. Bleiziffer, A. Heßelmann, A. Görling [J. Chem. Phys.](https://doi.org/10.1063/1.4818984) 139, 084113 (2013)  
[6] S. Fauser,  E. Trushin, A. Görling [J. Chem. Phys.](https://doi.org/10.1063/5.0263582) 162, 164108 (2025)
