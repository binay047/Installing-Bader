# 0.C. Installing Bader

## Theory

`bader` is a grid-based Bader charge analysis program developed by the Henkelman group. It is used to analyze the electron density of a material by partitioning the charge density into atomic basins using zero-flux surfaces.

For Quantum ESPRESSO calculations, the charge density is first converted into a Gaussian Cube file using `pp.x`. The resulting `.cube` file is then analyzed using the `bader` executable to obtain the charge associated with each atomic basin.

Unlike `thermo_pw`, Bader is a standalone program and does not need to be compiled inside the Quantum ESPRESSO source directory.

## Procedure

* Download the Linux 64-bit version:

  ```bash
  wget https://theory.cm.utexas.edu/henkelman/code/bader/download/bader_lnx_64.tar.gz
  ```

* Extract the downloaded archive:

  ```bash
  tar xzvf bader_lnx_64.tar.gz
  ```

* Create a personal binary directory:

  ```bash
  mkdir -p ~/bin
  ```

* Move the Bader executable to the binary directory:

  ```bash
  mv bader ~/bin/
  ```

* Add `~/bin` to your PATH:

  ```bash
  echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
  source ~/.bashrc
  ```

* Check the location of the Bader executable:

  ```bash
  which bader
  ```

  The output should look similar to:


>  /home/username/bin/bader


* Check the Bader version:

  ```bash
  bader -h
  ```

  The output should display the Bader program information and version.

* **Congratulations!!!** Bader is now installed.

## Using Bader with Quantum ESPRESSO

Bader analysis requires an electron-density file on a regular three-dimensional grid. For Quantum ESPRESSO calculations, the charge density can be converted into a Gaussian Cube file using `pp.x`.

### 1. Generate the Charge Density

Create a file named `pp.in`:

```text
&INPUTPP
  prefix = 'aiida'
  outdir = './out/'
  filplot = 'rho'
  plot_num = 0
/

&PLOT
  iflag = 3
  output_format = 6
  fileout = 'rho.cube'
/
```

Run:

```bash
pp.x < pp.in > pp.out
```

If the calculation is successful, the Gaussian Cube file will be generated:

```text
rho.cube
```

Check the generated file:

```bash
ls -lh rho.cube
```

### 2. Run Bader Analysis

Run:

```bash
bader -i cube rho.cube
```

The main output file is:

```text
ACF.dat
```

View the results:

```bash
cat ACF.dat
```

## Understanding `ACF.dat`

The main columns in `ACF.dat` are:

```text
#         X           Y           Z       CHARGE      MIN DIST   ATOMIC VOL
```

where:

* `X`, `Y`, `Z` are the coordinates of the atomic Bader basin.
* `CHARGE` is the number of electrons contained within the Bader basin.
* `MIN DIST` is the minimum distance between the atom and the Bader surface.
* `ATOMIC VOL` is the volume of the Bader basin.

At the bottom of `ACF.dat`, Bader reports:

```text
VACUUM CHARGE:
VACUUM VOLUME:
NUMBER OF ELECTRONS:
```

The `NUMBER OF ELECTRONS` should be approximately equal to the total number of valence electrons represented by the pseudopotentials used in the Quantum ESPRESSO calculation.

## Checking the Total Number of Electrons

The valence electron count can be checked directly from the Quantum ESPRESSO pseudopotential files:

```bash
grep -i "z_valence" pseudo/*.UPF
```

For example, for BeScCoSi:

```text
Be = 2
Sc = 11
Co = 17
Si = 4
```

Therefore:

```text
Total valence electrons = 2 + 11 + 17 + 4
                         = 34
```

The Bader calculation should therefore give approximately:

```text
NUMBER OF ELECTRONS:        34.0000
```

Small differences in the last decimal places can occur because of numerical integration.

## Calculating Bader Atomic Charges

The `CHARGE` column in `ACF.dat` gives the number of electrons inside each Bader basin.

The net Bader charge can be calculated using:

\[
q_i = Z_i^{\mathrm{valence}} - N_i^{\mathrm{Bader}}
\]

where:

* \(Z_i^{\mathrm{valence}}\) is the valence electron count of the pseudopotential.
* \(N_i^{\mathrm{Bader}}\) is the number of electrons in the Bader basin.

For example, for BeScCoSi:

```text
Be:
Zval   = 2.000000
NBader = 0.067046
```

Therefore:

```text
q(Be) = 2.000000 - 0.067046
      = +1.932954 e
```

A positive Bader charge indicates electron depletion relative to the pseudopotential valence reference, while a negative Bader charge indicates electron accumulation.

## Checking the Pseudopotential Type

The pseudopotential type can be checked using:

```bash
grep -i "pseudo_type\|is_ultrasoft\|is_paw\|z_valence" pseudo/*.UPF
```

For example, an ultrasoft pseudopotential may contain:

```text
pseudo_type="USPP"
is_ultrasoft="true"
is_paw="false"
```

A PAW pseudopotential contains:

```text
is_paw="true"
```

The pseudopotential type is important when interpreting Bader charges because the charge density near atomic cores depends on the pseudopotential formalism.

## Bader Grid Convergence

Bader analysis should be tested for convergence with respect to the charge-density grid.

First run Bader using the original charge-density grid:

```bash
bader -i cube rho.cube
cp ACF.dat ACF_original.dat
```

Then generate a finer charge-density grid using `pp.x` and repeat the Bader analysis:

```bash
bader -i cube rho_120.cube
cp ACF.dat ACF_120.dat
```

A further calculation can be performed using an even finer grid:

```bash
bader -i cube rho_180.cube
cp ACF.dat ACF_180.dat
```

The Bader charges should remain approximately unchanged when the grid resolution is increased.

For example:

```text
Grid       Be          Sc          Co          Si
--------------------------------------------------------
60³       0.067046    9.747083    17.963315   6.222177
120³      0.067046    9.747083    17.963315   6.222177
180³      0.067046    9.747083    17.963315   6.222177
```

If the values remain stable, the Bader charge partition is numerically converged with respect to the tested grid.

## Important Notes

* Always use a converged Quantum ESPRESSO charge density for Bader analysis.
* Check that the total number of electrons reported by Bader agrees with the expected valence electron count.
* Check Bader charge convergence by increasing the charge-density grid.
* Inspect the `MIN DIST` and `ATOMIC VOL` columns for unusual Bader basins.
* The `CHARGE` column represents the number of electrons in a Bader basin, not an oxidation state.
* Bader charges depend on the reference valence charge of the pseudopotential.
* For calculations using ultrasoft pseudopotentials (USPP), Bader charges should be interpreted carefully because the charge density near the atomic cores is represented within the pseudopotential formalism.
* PAW-based charge densities are generally preferable when high-quality Bader charge analysis is required.
* A numerically converged Bader result does not necessarily mean that the resulting charge has a unique chemical interpretation.

## Useful Commands

Check Bader installation:

```bash
which bader
```

Check Bader version:

```bash
bader -h
```

Generate the Cube file:

```bash
pp.x < pp.in > pp.out
```

Run Bader analysis:

```bash
bader -i cube rho.cube
```

View the Bader results:

```bash
cat ACF.dat
```

Check the total electron count:

```bash
grep -i "NUMBER OF ELECTRONS" ACF.dat
```

Check pseudopotential valence electrons:

```bash
grep -i "z_valence" pseudo/*.UPF
```

* **Congratulations!!!**
