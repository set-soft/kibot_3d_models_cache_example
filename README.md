# KiBot example of how to cache 3D models

The KiBot docker images doesn't contain KiCad's 3D models.
This is because they take a huge ammout of resources.

KiBot will try to download any model which name starts with
`${KISYS3DMOD}/` or `${KICAD6_3DMODEL_DIR}/` from the KiCad's
[repo](https://gitlab.com/kicad/libraries/kicad-packages3D/-/raw/master/).

The drawback is that you have to download them for each run.
But GitHub has a cache mechanism. This example shows how to use GitHub's cache
to avoid downloading the models on each CI/CD run.

For our example we'll use the following board:

![Example](images/light_control-3D_blender_top.png)

In this example all but two 3D models come from the KiCad lib.


## Using GitHub actions

The example workflow is [kibot_gha.yml](.github/workflows/kibot_gha.yml).
The most relevant parts are:

```yaml
    - name: Cache 3D models data
      id: models-cache
      uses: set-soft/cache@main
      with:
        path: ~/cache_3d
        key: cache_3d
```

This step stores/retrieves the content of `~/cache_3d`.

Then we use the `cache3D` option, like this:

```yaml
    - name: Run KiBot
      # full k6 + dev
      # You can also use v2_k6 when 1.5.2 is released
      uses: INTI-CMNB/KiBot@v2_dk6
      with:
        config: configs/blender_export.kibot.yaml
        dir: Generated
        board: pcb/light_control.kicad_pcb
        cache3D: YES
        verbose: 4
```

Note that the GHA works only with *~/cache_3d* as cache.


## Using a docker image

The example workflow is [kibot.yml](.github/workflows/kibot.yml).
The most relevant parts are:

```yaml
    - name: Cache 3D models data
      id: models-cache
      #uses: actions/cache@v3
      uses: set-soft/cache@main
      with:
        path: ~/cache_3d
        key: cache_3d
```

This step stores/retrieves the content of `~/cache_3d`.
We use the `set-soft/cache@main` because it can store the results even if the workflow fails.
But you can also use `actions/cache@v3`.

Then we just define the `KIBOT_3D_MODELS` environment variable in the same step that we run KiBot:

```yaml
    - name: Run KiBot
      run: |
        export KIBOT_3D_MODELS="$HOME/cache_3d"
        mkdir Generated
        kibot -vvvv -c configs/blender_export.kibot.yaml -b pcb/light_control.kicad_pcb -d Generated 2> Generated/kibot.log
```

The export is the relevant thing here.

## What we get

This is all we need, the first run will download the models, the next runs will use the cached models.
The cache is stored [here](https://github.com/set-soft/kibot_3d_models_cache_example/actions/caches).

If you take a look at KiBot logs, in this example stored in the artifacts, you'll see the first run
says:

```
DEBUG:Missing 3D model file ${KISYS3DMOD}/Resistor_SMD.3dshapes/R_0603_1608Metric.wrl (${KISYS3DMOD}/Resistor_SMD.3dshapes/R_0603_1608Metric.wrl) (kibot - out_base_3d.py:163)
DEBUG:Downloading `https://gitlab.com/kicad/libraries/kicad-packages3D/-/raw/master/Resistor_SMD.3dshapes/R_0603_1608Metric.wrl` (kibot - out_base_3d.py:101)
```

But the next run says:

```
DEBUG:Missing 3D model file ${KISYS3DMOD}/Resistor_SMD.3dshapes/R_0603_1608Metric.wrl (${KISYS3DMOD}/Resistor_SMD.3dshapes/R_0603_1608Metric.wrl) (kibot - out_base_3d.py:163)
DEBUG:Using cached model `/github/home/cache_3d/Resistor_SMD.3dshapes/R_0603_1608Metric.wrl` (kibot - out_base_3d.py:99)
```

## A special note

If you take a deep look at the [kibot_gha.yml](.github/workflows/kibot_gha.yml) file you'll notice it says:

```yaml
  KiBot:
    name: Render PCB
    runs-on: ubuntu-latest
    container: ghcr.io/inti-cmnb/kicad6_auto_full:dev
```

Here the `container` definition looks unnecessary. However, if you remove it, the cache won't be retrieved.
This is most probably because we are sharing the cache between the GHA and the regular docker image,
but I couldn't find a definitive answer in the docs.

