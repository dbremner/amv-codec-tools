# Origin #

ACT is a low complexity speech compression codec. There are 3 different ways to record to this codec from a player.

  * Fine Rec
  * Long Rec
  * Long Vox, in this mode the player will just record if there is anything to record.

| **Name** | **Sample rate** | **Frame size** | **Frames per chunk** | **Chunk size** | **Padding bytes** |
|:---------|:----------------|:---------------|:---------------------|:---------------|:------------------|
| Fine Rec | 8000            | 10             | 51                   | 510            | 2                 |
| Long Rec | 4400            | 22<sup>1</sup> | 23                   | 506            | 6                 |
| Long Vox | 4400            | 22<sup>1</sup> | 23                   | 506            | 6                 |
**Table 1. Known Formats**

<sup>1</sup> Each ACT frame contains two G.729 frames.

The codec actually is G.729A

# ACT File Format #

Files are always aligned to 512 bytes boundary and contains 512 byte header and chunks of the same size. Each chunk consists of fixed number of frames and padding bytes (see Table 1).

| **Name** | **Offset** | **Bytes** | **Description** |
|:---------|:-----------|:----------|:----------------|
| riff tag | 0          | 4         | 'RIFF'          |
| riff size | 4          | 4         | `data_size + 36` |
| wave tag | 8          | 4         | 'WAVE'          |
| fmt tag  | 12         | 4         | 'fmt '          |
| wave header size | 16         | 4         | Always 0x10     |
| wave format | 20         | 2         | PCM's 0x01      |
| channels | 22         | 2         | 0x01            |
| sample rate | 24         | 4         | 8000 or 4400    |
| bit rate | 28         | 4         | 16000 or ?      |
| block align | 32         | 2         | Always 0x02     |
| bits per sample | 34         | 2         | Always 16       |
| data\_tag | 36         | 4         | 'data'          |
| data\_size | 40         | 4         | `chunks_count *  8 * chunk_size`<sup>1</sup> |
| reserved | 44         | 212       | filled by zero  |
| act tag  | 256        | 1         | Always 0x84     |
| millisec | 257        | 2         | milliseconds part of duration value |
| seconds  | 259        | 1         | seconds part of duration value |
| minutes  | 260        | 4         | minutes part of duration value |
| reserved | 264        | 248       | filled by zero  |
**Table 2. ACT header**

<sup>1</sup> chunk\_size is determined from Table 1.

## ACT Frame format ##
First half of ACT frame contains odd bytes, while the second contains even bytes of encoded G.729 frame.
```
for(i=0 ; i<frame_size/2; i++)
{
  g729_data[2*i]=act_data[i]
  g729_data[2*i+1]=act_data[frame_size/2+i]
}
```

## G.729 Codec ##

| **Symbol** | **Description** | **Bits (Fine rec)** | **Bits (Long Rec/Long vox)** |
|:-----------|:----------------|:--------------------|:-----------------------------|
| L0         | Switched MA predictor of LSP quantizer | 1                   | 1                            |
| L1         | First stage vector of quantizer | 7                   | 7                            |
| L2         | Second stage lower vector of LSP quantizer | 5                   | 5                            |
| L3         | Second stage higher vector of LSP quantizer | 5                   | 5                            |
| P1         | Pitch delay first subframe | 8                   | 8                            |
| P0         | Parity bit for Pitch delay | 1                   | 1                            |
| C1         | Fixed codebook first subframe | 13                  | 12(17)<sup>2</sup>           |
| S1         | Signs of fixed-codebook pulses 1st subframe | 4                   | 9(4)<sup>2</sup>             |
| GA1        | Gain codebook (stage 1) 1st subframe | 3                   | 3                            |
| GB1        | Gain codebook (stage 1) 1st subframe | 4                   | 4                            |
| P2         | Pitch delay second subframe | 5                   | 5                            |
| C2         | Fixed codebook 2nd subframe | 13                  | 12(17)<sup>2</sup>           |
| S2         | Signs of fixed-codebook pulses 2nd subframe | 4                   | 9(4)<sup>2</sup>             |
| GA2        | Gain codebook (stage 1) 2nd subframe | 3                   | 3                            |
| GB2        | Gain codebook (stage 1) 2nd subframe | 4                   | 4                            |
|            | **Total**       | **80**              | **88**                       |
**Table 3. Description of transmitted parameters indices**<sup>1</sup>

<sup>1</sup>Note: the bitstream ordering is reflected by order in the table. For each parameter, the Most Significant Bit (MSB) is transmitted first.

<sup>2</sup>This affects fixed-codebook pulses indexes calculation. "Fine rec" mode uses 3+3+3+4 bit allocation which gives four pulses units with indexes in [0:39] range. Since indexes must cover [0:43] range for "Long rec" and "Long vox" formats, bit allocation was changed to 4+4+4+5=17 for them. And since proprietary codec uses 16-bit arithmetic last 5 bits were transferred to next parameter. New codec with 32-bit parameters will use 17-bit and 4-bit values for this and next parameters.

# Proprietary software #

|**Offset**| **Function**<sup>1</sup> | **Description** |
|:---------|:-------------------------|:----------------|
|0x00401000| ReadHeader               | Read ACT Header |
|0x00401030| ReadACTFrame             | Reads one (two for Long Rec/Vox) frame to buffer |
|0x00401130| SkipRiffHeader           |                 |
|0x00401150| WriteRiffHeader          | Writes Riff header to decoded WAV file|
|0x00401280| InitCodecParams          | Initializes codec|
|0x00407C90| ACT2WavConvert           | Main convertion routine (loop) |
|0x004029E0| DecodeACTFrame           | G.729 decoder   |
**Table 4. Proprietary decoder functions**

<sup>1</sup> Functions names are guessed

|**Offset**| **Function** | **Description** |
|:---------|:-------------|:----------------|
|0x004013A0| sature       |                 |
|0x004013F0| add          |                 |
|0x00401410| sub          |                 |
|0x00401430| shl          |                 |
|0x004014A0| shr          |                 |
|0x004014F0| mult         |                 |
|0x00401520| L\_mult      |                 |
|0x00401550| negate       |                 |
|0x00401570| extract\_h   |                 |
|0x00401580| extract\_l   |                 |
|0x00401590| round        |                 |
|0x004015B0| L\_mac       |                 |
|0x004015D0| L\_msu       |                 |
|0x004015F0| L\_add       |                 |
|0x00401630| L\_sub       |                 |
|0x00401670| L\_shl       |                 |
|0x004016D0| L\_shr       |                 |
|0x00401710| L\_deposit\_h |                 |
|0x00401720| L\_deposit\_l |                 |
|0x00401730| L\_shr\_r    |                 |
|0x00401770| div\_s       |                 |
|0x00401820| norm\_l      |                 |
**Table 4. Routines which were got from ITU's reference decoder (BASIC\_OP.C)**


# Links #
  * [ACT Page on wiki.multimedia.cx](http://wiki.multimedia.cx/index.php?title=ACT)
  * [ITU G.729 Specification](http://amv-codec-tools.googlecode.com/files/G.729_e.pdf)
  * [Samples](http://samples.mplayerhq.hu/A-codecs/act/)
  * [Working decoder](http://www.netac.com/Download/C635/C635%20Driver&Tools.zip)