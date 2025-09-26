# Game Plan

## Rustification

### Implement Rust `pyfunction`s for hot loops in `qiskit-addon-sqd`
- **`_bipartite_bitstring_correcting`**  
  - High call count, pure Python implementation.  
  - Excellent first target for Rust + Rayon parallelization.  
  - We need to maintain deterministic RNG per shot to ensure reproducibility.

- **`configuration_recovery`**  
  - Coordinates multiple passes of `_bipartite_*`.  
  - Wrapping this in Rust avoids Python round trips and will give us broader end-to-end speedups.  
  - Output must be NumPy arrays; we will parallelize across shots for scalability.

### Monkey-patch PySCF functions with Rust-backed drop-ins
- **`selected_ci.py`**
  - **`hop`**: Excitation generation; currently dominated by Python-level loops and deduplication.  
  - **`gen_map`**: Heavy reliance on `unique` and `argsort`. Replace with fused Rust deduplication + sorting.  
  - **`mk_rdm2`**: Nested accumulations; we can parallelize with Rayon to avoid Python bottlenecks.  
  - **`contract_ss`**: Hot contraction kernel; port for tighter memory control and loop efficiency.

- **`numpy_helpers.py` in PySCF**
  - Replace micro-kernels where Python dispatch dominates.  
  - Targets:  
    - `take_2d` / `takebak_2d` → vector gather/scatter.  
    - `_arraysetops_impl.unique`, `_unique1d`, `argsort`, `cumsum`, `sum` → fused unique/sort/reduce ops.

---

## Methods

- **Focus PRs on `qiskit_addon_sqd` repo**  
  - That is our primary project and the best place to demonstrate measurable wins.  
  - We should frame PRs as “opt-in backend acceleration” to minimize review friction.

- **Monkey-patch PySCF locally**  
  - Keep signatures identical to PySCF.  
  - Patch in Rust kernels only where bottlenecks are proven.  
  - Maintain the original Python implementation as a fallback.

- **Backend toggles**  
  - Allow users to switch explicitly (or fall back automatically if Rust is missing).  
  - Example:
    ```python
    def make_rdm2(..., backend="python"):
        if backend == "rust" and HAVE_RUST:
            return rust_make_rdm2(...)
        return _python_make_rdm2(...)
    ```

- **Testing discipline**  
  - Use small molecules (H₂, H₂O, N₂) as gold test cases.  
  - Unit tests should assert exact equivalence, or statistical match if RNG is involved.  
  - Benchmarks should clearly show improvements (e.g., “Python 300s → Rust 40s”).

- **Future-proofing**  
  - Preserve `kwargs` (even if unused).  
  - Document the `backend` argument; keep default as Python.  
  - This ensures contributions remain PR-friendly if we decide to upstream later.

# Notes
## Preliminary profiling on $N_2$.

```
Iteration 6
	Subsample 0
		Energy: -108.9298383856094
		Subspace dimension: 1600
	Subsample 1
		Energy: -108.9298383856094
		Subspace dimension: 1600
	Subsample 2
		Energy: -108.9298383856094
		Subspace dimension: 1600
Iteration 7
	Subsample 0
		Energy: -109.07888016408755
		Subspace dimension: 277729
	Subsample 1
		Energy: -109.0328160765794
		Subspace dimension: 272484
	Subsample 2
		Energy: -109.03091620296466
		Subspace dimension: 273529
Iteration 8
	Subsample 0
		Energy: -109.14343714924301
		Subspace dimension: 405769
	Subsample 1
		Energy: -109.14184224218769
		Subspace dimension: 405769
	Subsample 2
		Energy: -109.12670411823623
		Subspace dimension: 412164
Iteration 9
	Subsample 0
		Energy: -109.1590921137567
		Subspace dimension: 597529
	Subsample 1
		Energy: -109.17261756698423
		Subspace dimension: 595984
	Subsample 2
		Energy: -109.17070524484906
		Subspace dimension: 583696
Iteration 10
	Subsample 0
		Energy: -109.17725549896457
		Subspace dimension: 799236
	Subsample 1
		Energy: -109.17671083790574
		Subspace dimension: 795664
	Subsample 2
		Energy: -109.1817113343019
		Subspace dimension: 806404
         178418083 function calls (164006196 primitive calls) in 4279.188 seconds

   Ordered by: cumulative time
   List reduced from 462 to 60 due to restriction <60>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        5    0.000    0.000 4036.661  807.332 fermion.py:435(solve_sci_batch)
       15    0.003    0.000 3642.952  242.863 fermion.py:476(solve_sci)
       15    0.002    0.000 2817.111  187.807 selected_ci.py:313(kernel_fixed_space)
      108    0.001    0.000 2811.756   26.035 direct_spin1.py:926(<lambda>)
      108    0.002    0.000 2458.449   22.763 selected_ci.py:346(hop)
      108    5.364    0.050 2458.440   22.763 addons.py:534(contract_2e)
       15    1.090    0.073  608.232   40.549 selected_ci.py:830(make_rdm2)
   716899   20.323    0.000  331.563    0.000 ipython-input-3299003139.py:15(_snapshot)
       15    0.001    0.000  263.628   17.575 direct_spin1.py:920(eig)
   400000   93.985    0.000  192.766    0.000 configuration_recovery.py:181(_bipartite_bitstring_correcting)
1446620/1445829    5.943    0.000   96.052    0.000 compilerop.py:181(check_linecache_ipython)
1446620/1445829   13.859    0.000   78.688    0.000 linecache.py:52(checkcache)
1446611/1445841   44.973    0.000   49.103    0.000 {built-in method posix.stat}
      108    0.026    0.000   48.739    0.451 selected_ci.py:809(contract_ss)
  1520758    4.223    0.000   38.062    0.000 _arraysetops_impl.py:144(unique)
  1520758   20.717    0.000   30.557    0.000 _arraysetops_impl.py:342(_unique1d)
1116931/1116929    9.561    0.000   26.252    0.000 {method 'join' of 'str' objects}
  3156077    7.999    0.000   24.644    0.000 fromnumeric.py:69(_wrapreduction)
  2378033    4.315    0.000   21.290    0.000 fromnumeric.py:2255(sum)
    11232   14.443    0.001   14.603    0.001 numpy_helper.py:506(takebak_2d)
  3156278   13.780    0.000   13.780    0.000 {method 'reduce' of 'numpy.ufunc' objects}
  4322764    4.320    0.000   12.269    0.000 traceback.py:391(extended_frame_gen)
  1520695    2.253    0.000   12.115    0.000 fromnumeric.py:2609(cumsum)
   716899   10.747    0.000   10.747    0.000 {built-in method sys._current_frames}
  3584500    7.452    0.000   10.721    0.000 ipython-input-3299003139.py:20(<genexpr>)
  1446758   10.461    0.000   10.461    0.000 {method 'update' of 'dict' objects}
  8476196    5.386    0.000   10.131    0.000 configuration_recovery.py:162(_p_flip_1_to_0)
  1520972    2.041    0.000    9.892    0.000 fromnumeric.py:51(_wrapfunc)
   778044    1.348    0.000    9.786    0.000 fromnumeric.py:3068(prod)
 20800000    9.763    0.000    9.763    0.000 configuration_recovery.py:131(_p_flip_0_to_1)
  4322764    7.950    0.000    7.950    0.000 traceback.py:327(walk_stack)
  1520695    7.235    0.000    7.235    0.000 {method 'cumsum' of 'numpy.ndarray' objects}
3601591/3601587    5.911    0.000    6.019    0.000 traceback.py:265(__init__)
 21200000    5.976    0.000    5.976    0.000 configuration_recovery.py:122(<genexpr>)
  3601591    4.295    0.000    5.961    0.000 linecache.py:152(lazycache)
  1520740    5.287    0.000    5.287    0.000 {method 'argsort' of 'numpy.ndarray' objects}
    11340    4.857    0.000    5.233    0.000 numpy_helper.py:478(take_2d)
 13797993    5.018    0.000    5.018    0.000 {built-in method builtins.len}
   778028    1.979    0.000    4.736    0.000 numerictypes.py:470(issubdtype)
  2867600    3.269    0.000    3.269    0.000 {method 'rsplit' of 'str' objects}
  3601594    3.104    0.000    3.104    0.000 {method 'strip' of 'str' objects}
       15    0.028    0.002    3.035    0.202 linalg_helper.py:291(davidson1)
   782290    2.744    0.000    2.744    0.000 {built-in method builtins.round}
  1556056    1.421    0.000    2.607    0.000 numerictypes.py:288(issubclass_)
  2301000    2.459    0.000    2.459    0.000 {built-in method builtins.getattr}
  3602271    2.214    0.000    2.214    0.000 {method 'append' of 'list' objects}
  1521568    2.201    0.000    2.201    0.000 {built-in method numpy.empty}
      108    0.180    0.002    2.154    0.020 selected_ci.py:623(contract_ss)
  1520762    2.085    0.000    2.085    0.000 {method 'flatten' of 'numpy.ndarray' objects}
  3601591    2.054    0.000    2.054    0.000 {method 'add' of 'set' objects}
  1556056    1.051    0.000    1.771    0.000 getlimits.py:487(__new__)
       30    0.003    0.000    1.736    0.058 selected_ci.py:812(make_rdm1s)
   716899    1.709    0.000    1.709    0.000 {method 'keys' of 'dict' objects}
      554    1.694    0.003    1.694    0.003 {built-in method builtins.sorted}
      262    1.493    0.006    1.493    0.006 {built-in method numpy.array}
  1520758    0.867    0.000    1.341    0.000 _arraysetops_impl.py:131(_unpack_tuple)
  2334248    1.336    0.000    1.336    0.000 {built-in method builtins.issubclass}
      432    1.161    0.003    1.163    0.003 selected_ci.py:631(gen_map)
  3156170    0.999    0.000    0.999    0.000 {method 'items' of 'dict' objects}
   801797    0.973    0.000    0.973    0.000 {built-in method numpy.zeros}




Top caller→callee edges (by approx cumulative time):
 4036.661s  _spy → solve_sci_batch  (5)  [ipython-input-3299003139.py:58 → fermion.py:435]
 3642.952s  solve_sci_batch → solve_sci  (15)  [fermion.py:435 → fermion.py:476]
 2817.111s  solve_sci → kernel_fixed_space  (15)  [fermion.py:476 → selected_ci.py:313]
 2703.508s  _spy → <lambda>  (97)  [ipython-input-3299003139.py:58 → direct_spin1.py:926]
 2458.449s  <lambda> → hop  (108)  [direct_spin1.py:926 → selected_ci.py:346]
  609.527s  solve_sci → _spy  (107)  [fermion.py:476 → ipython-input-3299003139.py:58]
  608.232s  solve_sci → make_rdm2  (15)  [fermion.py:476 → selected_ci.py:830]
  535.823s  contract_2e → _spy  (282613)  [addons.py:534 → ipython-input-3299003139.py:58]
  351.733s  <lambda> → _spy  (45)  [direct_spin1.py:926 → ipython-input-3299003139.py:58]
  297.510s  eig → _spy  (41)  [direct_spin1.py:920 → ipython-input-3299003139.py:58]
  261.755s  _spy → eig  (13)  [ipython-input-3299003139.py:58 → direct_spin1.py:920]
  208.917s  _spy → _spy  (11946)  [ipython-input-3299003139.py:58 → ipython-input-3299003139.py:58]
   81.876s  <built-in method time.sleep> → _spy  (176405)  [~:0 → ipython-input-3299003139.py:58]
   70.198s  davidson1 → <lambda>  (9)  [linalg_helper.py:291 → direct_spin1.py:926]
   49.179s  make_rdm2 → _spy  (170276)  [selected_ci.py:830 → ipython-input-3299003139.py:58]
   48.739s  contract_2e → contract_ss  (108)  [addons.py:534 → selected_ci.py:809]
   42.085s  _spy → _bipartite_bitstring_correcting  (356287)  [ipython-input-3299003139.py:58 → configuration_recovery.py:181]
   38.050s  <built-in method time.sleep> → <lambda>  (2)  [~:0 → direct_spin1.py:926]
   31.060s  contract_2e → _snapshot  (280985)  [addons.py:534 → ipython-input-3299003139.py:15]
   20.409s  _bipartite_bitstring_correcting → sum  (2316372)  [configuration_recovery.py:181 → fromnumeric.py:2255]
   16.205s  sum → _wrapreduction  (2378033)  [fromnumeric.py:2255 → fromnumeric.py:69]
   15.456s  _snapshot → <method 'join' of 'str' objects>  (728709)  [ipython-input-3299003139.py:15 → ~:0]
   12.269s  _extract_from_extended_frame_gen → extended_frame_gen  (4322748)  [traceback.py:399 → traceback.py:391]
   11.949s  checkcache → _bipartite_bitstring_correcting  (25557)  [linecache.py:52 → configuration_recovery.py:181]
   10.836s  _bipartite_bitstring_correcting → cumsum  (1405228)  [configuration_recovery.py:181 → fromnumeric.py:2609]
   10.747s  _snapshot → <built-in method sys._current_frames>  (716899)  [ipython-input-3299003139.py:15 → ~:0]
   10.721s  <method 'join' of 'str' objects> → <genexpr>  (3584500)  [~:0 → ipython-input-3299003139.py:20]
   10.434s  check_linecache_ipython → <method 'update' of 'dict' objects>  (1442968)  [compilerop.py:181 → ~:0]
    9.862s  cumsum → _wrapfunc  (1520695)  [fromnumeric.py:2609 → fromnumeric.py:51]
    9.679s  _spy → <method 'join' of 'str' objects>  (358359)  [ipython-input-3299003139.py:58 → ~:0]
    8.053s  _snapshot → _bipartite_bitstring_correcting  (15286)  [ipython-input-3299003139.py:15 → configuration_recovery.py:181]
    7.950s  extended_frame_gen → walk_stack  (4322764)  [traceback.py:391 → traceback.py:327]
    7.235s  _wrapfunc → <method 'cumsum' of 'numpy.ndarray' objects>  (1520695)  [fromnumeric.py:51 → ~:0]
    6.019s  _extract_from_extended_frame_gen → __init__  (3601578)  [traceback.py:399 → traceback.py:265]
    5.961s  _extract_from_extended_frame_gen → lazycache  (3601578)  [traceback.py:399 → linecache.py:152]
    4.832s  <built-in method time.sleep> → _snapshot  (175396)  [~:0 → ipython-input-3299003139.py:15]
    4.744s  _p_flip_1_to_0 → _p_flip_0_to_1  (8476194)  [configuration_recovery.py:162 → configuration_recovery.py:131]
    4.505s  contract_ss → _spy  (6496)  [selected_ci.py:809 → ipython-input-3299003139.py:58]
    3.269s  <genexpr> → <method 'rsplit' of 'str' objects>  (2867600)  [ipython-input-3299003139.py:20 → ~:0]
    3.104s  line → <method 'strip' of 'str' objects>  (3601592)  [traceback.py:318 → ~:0]

Saved call-graph edges → prof_edges.csv

Top 25 sampled stacks (interval=0.005s, samples=716900):

[100.0%] 716900 samples
threading.py:1032 _bootstrap | threading.py:1075 _bootstrap_inner | threading.py:1012 run | ipython-input-3299003139.py:27 _run

SleepSpy: sleep seconds histogram: [(1.0, 4274)]

SleepSpy: top call sites:
  4274  /usr/local/lib/python3.12/dist-packages/ipykernel/parentpoller.py:37 run
  ```
