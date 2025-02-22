## Quick summary

|Instruction|General theme|Writemask|Optional special features|
|---|---|---|---|
|`vecint`&nbsp;(47≠4)|<code>z[_][i]&nbsp;±=&nbsp;f(x[i],&nbsp;y[i])</code>|9 bit|Indexed X or Y, shuffle X, shuffle Y,<br/>broadcast Y element, right shift, `sqrdmlah`|

## Instruction encoding

|Bit|Width|Meaning|Notes|
|---:|---:|---|---|
|10|22|[A64 reserved instruction](aarch64.md)|Must be `0x201000 >> 10`|
|5|5|Instruction|Must be `18`|
|0|5|5-bit GPR index|See below for the meaning of the 64 bits in the GPR|

## Operand bitfields

|Bit|Width|Meaning|Notes|
|---:|---:|---|---|
|(47=4) 63|1|Z is signed (`1`) or unsigned (`0`)|
|(47≠4) 63|1|X is signed (`1`) or unsigned (`0`)|
|58|5|Right shift amount|Ignored when ALU mode in {5, 6}|
|57|1|Ignored|
|54|3|Must be zero|No-op otherwise|
|53|1|[Indexed load](RegisterFile.md#indexed-loads) (`1`) or regular load (`0`)|
|(53=1) 52|1|Ignored|
|(53=1) 49|3|Register to index into|
|(53=1) 48|1|Indices are 4 bits (`1`) or 2 bits (`0`)|
|(53=1) 47|1|Indexed load of Y (`1`) or of X (`0`)|
|(53=0) 47|6|ALU mode|
|46|1|Ignored|
|42|4|Lane width mode|Meaning dependent upon ALU mode|
|41|1|Ignored|
|38|3|Write enable or broadcast mode|
|32|6|Write enable value or broadcast lane index|Meaning dependent upon associated mode|
|31|1|Ignored|
|(47=4) 30|1|Saturate Z (`1`) or truncate Z (`0`)|
|(47=4) 29|1|Right shift is rounding (`1`) or truncating (`0`)|
|(47≠4) 29|2|[X shuffle](RegisterFile.md#shuffles)|
|27|2|[Y shuffle](RegisterFile.md#shuffles)|
|(47=4) 26|1|Z saturation is signed (`1`) or unsigned (`0`)|
|(47≠4) 26|1|Y is signed (`1`) or unsigned (`0`)|
|20|6|Z row|Low bits ignored in some lane width modes|
|19|1|Ignored|
|10|9|X offset|
|9|1|Ignored|
|0|9|Y offset|

ALU modes:
|Integer operation|47|Notes|
|---|---|---|
|`z+((x*y)>>s)`|`0`|
|`z-((x*y)>>s)`|`1`|
|`z+((x+y)>>s)`|`2`|Particular write enable mode can skip `x` or `y`|
|`z-((x+y)>>s)`|`3`|Particular write enable mode can skip `x` or `y`|
|`z>>s` or `sat(z>>s)`|`4`|Shift can be rounding, saturation is optional|
|`sat(z+((x*y*2)>>16))`|`5`|Shift is rounding, saturation is signed|
|`sat(z-((x*y*2)>>16))`|`6`|Shift is rounding, saturation is signed|
|no-op|anything else|

When ALU mode < 4, lane width modes:
|X|Y|Z|42|
|---|---|---|---|
|i16 or u16|i16 or u16|i32 or u32 (two rows, interleaved pair)|`3`|
|i8 or u8|i8 or u8|i32 or u32 (four rows, interleaved quartet)|`10`|
|i8 or u8|i8 or u8|i16 or u16 (two rows, interleaved pair)|`11`|
|i8 or u8|i16 or u16 (each lane used twice)|i32 or u32 (four rows, interleaved quartet)|`12`|
|i16 or u16 (each lane used twice)|i8 or u8|i32 or u32 (four rows, interleaved quartet)|`13`|
|i16 or u16|i16 or u16|i16 or u16 (one row)|anything else|

When ALU mode = 4, lane width modes:
|Z |Z saturation|42|
|---|---|---|
|i32 or u32 (one row)|i16 or u16|`3`|
|i32 or u32 (one row)|i32 or u32|`4`|
|i8 or u8 (one row)|i8 or u8|`9`|
|i32 or u32 (one row)|i8 or u8|`10`|
|i16 or u16 (one row)|i8 or u8|`11`|
|i16 or u16 (one row)|i16 or u16|anything else|

When ALU mode > 4, lane width modes:

|X|Y|Z|42|
|---|---|---|---|
|i16 or u16|i16 or u16|i16 or u16 (one row)|anything|

Write enable or broadcast modes:
|Mode|Meaning of value (N)|
|---:|---|
|`0`|Enable all lanes (`0`), or odd lanes only (`1`), or even lanes only (`2`), or enable all lanes but override the ALU operation to `0` (`3`) or enable all lanes but override X values to `0` (`4`) or enable all lanes but override Y values to `0` (`5`) or no lanes enabled (anything else)|
|`1`|Enable all lanes, but broadcast Y lane #N to all lanes of Y|
|`2`|Only enable the first N lanes, or all lanes when N is zero|
|`3`|Only enable the last N lanes, or all lanes when N is zero|
|`4`|Only enable the first N lanes (no lanes when Z is zero)|
|`5`|Only enable the last N lanes (no lanes when Z is zero)|
|`6`|No lanes enabled|
|`7`|No lanes enabled|

## Description

When 47=4, performs an in-place reduction of an integer vector from Z, where reduction means right shift (either rounding or truncating), optionally followed by saturation (to i8/u8/i16/u16/i32/u32). Z values are 8 bit or 16 bit or 32 bit integers.

When 47≠4, performs some ALU operation between an X vector, a Y vector, and a Z vector, accumulating onto Z. Various combinations of line widths are permitted. When X or Y are narrower than Z, then multiple Z rows are used (as required to get the total number of Z elements to equal the number of X or Y elements). When this results in four Z rows, the data layout is:

<table><tr><th>Z0</th><td>0</td><td>4</td><td>8</td><td>12</td><td>16</td><td>20</td><td>24</td><td>28</td><td>32</td><td>36</td><td>40</td><td>44</td><td>48</td><td>52</td><td>56</td><td>60</td></tr>
<tr><th>Z1</th><td>1</td><td>5</td><td>9</td><td>13</td><td>17</td><td>21</td><td>25</td><td>29</td><td>33</td><td>37</td><td>41</td><td>45</td><td>49</td><td>53</td><td>57</td><td>61</td></tr>
<tr><th>Z2</th><td>2</td><td>6</td><td>10</td><td>14</td><td>18</td><td>22</td><td>26</td><td>30</td><td>34</td><td>38</td><td>42</td><td>46</td><td>50</td><td>54</td><td>58</td><td>62</td></tr>
<tr><th>Z3</th><td>3</td><td>7</td><td>11</td><td>15</td><td>19</td><td>23</td><td>27</td><td>31</td><td>35</td><td>39</td><td>43</td><td>47</td><td>51</td><td>55</td><td>59</td><td>63</td></tr></table>

## Emulation code

See [vecint.c](vecint.c).

A representative sample is:
```c
void emulate_AMX_VECINT(amx_state* state, uint64_t operand) {
    if ((operand >> 54) & 7) {
        return;
    }

    uint64_t z_row = (operand >> 20) & 63;
    int32_t omask = (((operand >> 32) & 0x1ff) == 3) ? 0 : -1;
    bool broadcast_y = ((operand >> (32+6)) & 7) == 1;
    int alumode = (operand & VECINT_INDEXED_LOAD) ? 0 : (operand >> 47) & 0x3f;
    uint32_t shift = (operand >> 58) & 0x1f;

    uint32_t xbits = 0, ybits = 0, zbits, satbits;
    if (alumode == 4) {
        switch ((operand >> 42) & 0xf) {
        case  3: zbits = 32; satbits = 16; break;
        case  4: zbits = 32; satbits = 32; break;
        case  9: zbits =  8; satbits =  8; break;
        case 10: zbits = 32; satbits =  8; break;
        case 11: zbits = 16; satbits =  8; break;
        default: zbits = 16; satbits = 16; break;
        }
    } else if (alumode == 5 || alumode == 6) {
        xbits = 16; ybits = 16; zbits = 16;
        shift = 15;
    } else {
        switch ((operand >> 42) & 0xf) {
        case  3: xbits = 16; ybits = 16; zbits = 32; break;
        case 10: xbits =  8; ybits =  8; zbits = 32; break;
        case 11: xbits =  8; ybits =  8; zbits = 16; break;
        case 12: xbits =  8; ybits = 16; zbits = 32; break;
        case 13: xbits = 16; ybits =  8; zbits = 32; break;
        default: xbits = 16; ybits = 16; zbits = 16; break;
        }
    }
    uint32_t xbytes = xbits / 8;
    uint32_t ybytes = ybits / 8;
    uint32_t zbytes = zbits / 8;

    if (alumode == 4) {
        ...
        return;
    } else if (alumode >= 7) {
        return;
    }

    uint8_t x[64];
    uint8_t y[64];
    load_xy_reg(x, state->x, (operand >> 10) & 0x1FF);
    load_xy_reg(y, state->y, operand & 0x1FF);
    if (operand & VECINT_INDEXED_LOAD) {
        uint32_t src_reg = (operand >> 49) & 7;
        uint32_t ibits = (operand & VECINT_INDEXED_LOAD_4BIT) ? 4 : 2;
        if (operand & VECINT_INDEXED_LOAD_Y) {
            load_xy_reg_indexed(y, state->y[src_reg].u8, ibits, ybits);
        } else {
            load_xy_reg_indexed(x, state->x[src_reg].u8, ibits, xbits);
        }
    }
    xy_shuffle(x, (operand >> 29) & 3, xbytes);
    xy_shuffle(y, (operand >> 27) & 3, ybytes);

    uint64_t x_enable = parse_writemask(operand >> 32, xbytes, 9);
    uint64_t y_enable = parse_writemask(operand >> 32, ybytes, 9);
    if (broadcast_y) {
        x_enable = ~(uint64_t)0;
        y_enable = ~(uint64_t)0;
    } else if (((operand >> (32+6)) & 7) == 0) {
        uint32_t val = (operand >> 32) & 0x3F;
        if (val == 4) {
            memset(x, 0, 64);
        } else if (val == 5) {
            memset(y, 0, 64);
        }
    }

    uint32_t xsignext = (operand & VECINT_SIGNED_X) ? (64 - xbits) : 0;
    uint32_t ysignext = (operand & VECINT_SIGNED_Y) ? (64 - ybits) : 0;
    uint32_t zsignext = 64 - zbits;
    uint32_t step = min(xbytes, ybytes);
    uint32_t zmask = (zbytes / step) - 1;
    for (uint32_t i = 0; i < 64; i += step) {
        uint32_t xi = i & -xbytes;
        if (!((x_enable >> xi) & 1)) continue;
        uint32_t yj = broadcast_y ? ((operand >> 32) * ybytes) & 0x3f : i & -ybytes;
        if (!((y_enable >> yj) & 1)) continue;
        int64_t xv = load_int(x + xi, xbytes, xsignext);        
        int64_t yv = load_int(y + yj, ybytes, ysignext);
        void* z = &state->z[bit_select(z_row, i / step, zmask)].u8[i & -zbytes];
        int64_t zv = load_int(z, zbytes, zsignext);
        int64_t result = vecint_alu(xv, yv, zv, alumode, shift) & omask;
        store_int(z, zbytes, result);
    }
}

int64_t vecint_alu(int64_t x, int64_t y, int64_t z, int alumode, uint32_t shift) {
    int64_t val = x * y;
    if (alumode == 5 || alumode == 6) {
        val += 1ull << (shift - 1);
    } else if (alumode == 2 || alumode == 3) {
        val = x + y;
    }
    val >>= shift;
    if (alumode == 1 || alumode == 3 || alumode == 6) {
        val = -val;
    }
    val += z;
    if (alumode == 5 || alumode == 6) {
        if (val > 32767) val = 32767;
        if (val < -32768) val = -32768;
    }
    return val;
}
```
