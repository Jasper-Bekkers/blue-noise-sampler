blue-noise-sampler
========
[![hassle on travis-ci.com](https://travis-ci.com/Jasper-Bekkers/blue-noise-sampler.svg?branch=master)](https://travis-ci.com/Jasper-Bekkers/blue-noise-sampler)
[![Latest version](https://img.shields.io/crates/v/blue-noise-sampler.svg)](https://crates.io/crates/blue-noise-sampler)
[![Documentation](https://docs.rs/blue-noise-sampler/badge.svg)](https://docs.rs/blue-noise-sampler)
![MIT](https://img.shields.io/badge/license-MIT-blue.svg)

This crate provides a rust version of the [A Low-Discrepancy Sampler that Distributes Monte Carlo Errors as a Blue Noise in Screen Space](https://eheitzresearch.wordpress.com/762-2/) sample code.

- [Documentation](https://docs.rs/blue-noise-sampler)

## Usage

Basically this crate just exposes a bunch of tables that can be uploaded to the GPU in a buffer, and then sampled from with something like the following HLSL code. This code is taken directly from the Sampler Code provided with the paper (same as the tables).

```hlsl
StructuredBuffer<int> g_blueNoiseSobol : register(t6, space0);
StructuredBuffer<int> g_blueNoiseScrambleTile : register(t7, space0);
StructuredBuffer<int> g_blueNoiseRankingTile : register(t8, space0);

float samplerBlueNoise(int pixel_i, int pixel_j, int sampleIndex, int sampleDimension)
{
	// wrap arguments
	pixel_i = pixel_i & 127;
	pixel_j = pixel_j & 127;
	sampleIndex = sampleIndex & 255;
	sampleDimension = sampleDimension & 255;

	// xor index based on optimized ranking
    // jb: 1spp blue noise has all 0 in g_blueNoiseRankingTile so we can skip the load
	int rankedSampleIndex = sampleIndex ^ g_blueNoiseRankingTile[sampleDimension + (pixel_i + pixel_j*128)*8];

	// fetch value in sequence
	int value = g_blueNoiseSobol[sampleDimension + rankedSampleIndex*256];

	// If the dimension is optimized, xor sequence value based on optimized scrambling
	value = value ^ g_blueNoiseScrambleTile[(sampleDimension%8) + (pixel_i + pixel_j*128)*8];

	// convert to float and return
	float v = (0.5f+value)/256.0f;
	return v;
}
```

There are some observations to be made, each module in the crate (`ssp1`, `spp2` etc) contain 3 global variables: `SOBOL`, `SCRAMBLING_TILE` and `RANKING_TILE` named identially so you can just `use blue_noise_sampler::spp16::*;` the right one in your code, and change it whenever you need another number of samples without updating the rest.

In the case of `spp1` the `RANKING_TILE` table is all 0's and so it doesn't strictly need to be uploaded to the GPU and the above code can be modified to do one load less as a minor optimization.

## License

Licensed under MIT license ([LICENSE](LICENSE) or http://opensource.org/licenses/MIT)


## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this crate by you, shall be licensed as above, without any additional terms or conditions.

Contributions are always welcome; please look at the [issue tracker](https://github.com/Jasper-Bekkers/blue-noise-sampler/issues) to see what known improvements are documented.
