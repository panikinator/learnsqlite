+++
title = 'Python BMP parser writer'
date = 2024-09-12T22:39:50+05:30
draft = false
+++

## python bmp parser and writer
```python
from decimal import DivisionByZero
from struct import pack, unpack_from, unpack
DivisionByZero()
def read_bmp(fp):
    f = open(fp, "rb")
    d = unpack_from("<HLLLLLLHHLLLLLL",f.read(54))
    width = d[5]
    height = d[6]
    f.seek(d[3])

    extra_bytes = ((4 - ((width*3) % 4)) % 4)
    if extra_bytes == 4:
        extra_bytes = 0

    rows = []
    for i in range(height):
        row = []
        for j in range(width):
            b = int.from_bytes(f.read(1), "little")
            g = int.from_bytes(f.read(1), "little")
            r = int.from_bytes(f.read(1), "little")
            pix = [r, g, b]
            row.append(pix)
        f.read(extra_bytes)
        rows.append(row)
    rows.reverse()
    f.close()

    return rows


def write_bmp(fp, data: list):
    data.reverse()
    # header
    signature = 19778 # H
    file_size = 0 # L
    reserved = 0 # L
    offset	 = 54 # L

    # info header
    info_header_size = 40 # L
    width = len(data[0]) # L
    height = len(data) # L
    planes = 1 # H
    num_bits_pix = 24 # H
    compression = 0 # L
    image_size = 0 # L
    XpixelsPerM = 100 #L
    YpixelsPerM = 100 #L
    colors_used = 255 # L
    imp_colors = 0 # L

    # HLLL
    # LLLHHLLLLLL
    # HLLLLLLHHLLLLLL

    extra_bytes = ((4 - ((width*3) % 4)) % 4)
    # extra_bytes = (bcWidth*3) % 4
    # extra_bytes= 4-((bcWidth*3) % 4)

    # if extra_bytes == 4:
    #     extra_bytes = 0
    file_size = offset+width*3*height + extra_bytes*height

    f = open(fp, "wb")

    f.write(pack('<HLLLLLLHHLLLLLL',
                    signature,
                    file_size,
                    reserved,
                    offset,
                    
                    info_header_size,
                    width,
                    height,
                    planes,
                    num_bits_pix,
                    compression,
                    image_size,
                    XpixelsPerM,
                    YpixelsPerM,
                    colors_used,
                    imp_colors
        ))

    for row in data:
        for pix in row:
            f.write(pack('<BBB', pix[2], pix[1], pix[0]))
        for i in range(extra_bytes):
            f.write(pack('B',0))
        # f.write(b"\0"*extra_bytes)
    f.close()

```
