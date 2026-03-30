# Half Dust Half Deity

## Description

A poet once said we are two things at once, the ground beneath our feet and the sky above our heads. They left this poem behind as proof. Read every word. Then read between them.

## Writeup

Opening it in vsc shows the characters in between the text.

1. `U+200B` ZERO WIDTH SPACE
2. `U+200D` ZERO WIDTH JOINER
3. `U+202A` LEFT-TO-RIGHT EMBEDDING
4. `U+202D` LEFT-TO-RIGHT OVERRIDE
5. `U+2063` INVISIBLE SEPARATOR

Each group of 7 zero width characters becomes one Unicode character.

```python
from pathlib import Path

text = Path("poetry.txt").read_text("utf-8")

alphabet = ['\u200b', '\u200d', '\u202a', '\u202d', '\u2063']
index = {ch: str(i) for i, ch in enumerate(alphabet)}

payload = [ch for ch in text if ch in index]

decoded = []
for i in range(0, len(payload), 7):
    chunk = payload[i:i+7]
    if len(chunk) < 7:
        break
    base5 = ''.join(index[ch] for ch in chunk)
    decoded.append(chr(int(base5, 5)))

print(''.join(decoded))
```

```bash
Where am I?


hackzero{9e82e9e902fb1b437230df2e66586433}
```

## Flag

`hackzero{9e82e9e902fb1b437230df2e66586433}`
