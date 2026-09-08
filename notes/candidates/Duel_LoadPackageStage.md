## `Duel_LoadPackageStage` at 0x800171A8

`gcc_2_8_1_g8_split`, 235 instructions against a target of 237, **two
instructions short**, both at one cross-jumping boundary. Written from scratch:
no stored candidate, no rows in `external_attempts.csv`.

The thirteen-stage callback registered with `File_RequestAsyncTransfer` by
`func_8001798C` and `func_800179F4` for the 235-sector duel terrain package.
Each stage fills the transfer descriptor for the next chunk: stages 1, 5, 7, 8,
9 and 11 point it at a fixed buffer and set the size, stages 3 to 5 hand over
the equip, fusion and ritual tables, stages 2 and 6 push a VRAM strip through
`LoadImage2` first, and stages 0, 6, 10 and 12 also set the destination
rectangle and mark the descriptor as an image stage. Every stage clears one of
`0x20000` / `0x30000` out of `D_8009B0F4` and writes 1 or 2 to `done`.

Integration will need three `split.yaml` rodata lines, because the file owns
`jtbl_800100C0` at file offset 0x8C0: `initial_data_0a` has to be cut at 0x8C0,
`[0x8C0, .rodata, game/<file>]` added, and the trailing alignment word at 0x8F4
left to a new `initial_data_0a?` segment - the pad after the last table belongs
to the following segment, not to this file.

It also needs two named fields in `file_transfer.h`, splitting `pad_00[0x8]`
into `pad_00[0x4]` plus `field_04`/`field_06` and `pad_32[0x6]` into `field_32`
plus `pad_34[0x4]`. Both are layout-preserving, the existing offset assertions
still hold, and no source names either pad.

### What is left

Two instructions at the shared tail that stages 0 and 10 use. Retail cross-jumps
them onto `[sw mode, sw value_08, addiu 0x800, sw value_0C]` and keeps the mode
constant in each arm, which works because stage 0 needs a `lui` for `0x20000`
and stage 10 an `addiu` for `0x4000`, so the backward scan stops there. In the
build, stage 10 instead merges onto the *image* tail that stages 6 and 12 share,
matching only its last three instructions.

Everything else lines up. The other shared tail, the one stages 6 and 12 use,
took three tries to place: written as duplicated straight-line code the build
came out three instructions long, and written as a `goto` with the whole tail
shared it came out five short. The stored source is the middle form: each arm
computes `flags = D_8009B0F4 & 0xFFDDFFFF;` and jumps to a label that starts at
the `field_04` store, which is exactly where retail's `.L800174F8` begins.

Tried for stage 10 and rejected: the same `goto` treatment for stages 0 and 10
(228, gcc then merges the whole `|= 0x10000` sequence as well), hoisting the
`D_8009B118` read into each arm (230), and six statement orders inside the two
stages.

### Levers that got it here

- **`D_8009B0F4` is `volatile` and needs the `.data` attribute.** Volatile is
  what keeps every read-modify-write separate, as retail has them; the section
  attribute is what keeps it out of small data at `-G8` so each access gets its
  own `lui`, which is the assembler macro form retail uses. A plain extern comes
  out `%gp_rel` and the function is twenty instructions short.
- **`-msplit-addresses` is required.** The jump-table dispatch is `lui`,
  `addiu`, `sll`, `addu`, `lw` - five instructions. Without the flag the
  assembler's `$at` macro does it in four.
- **`d->value_08 = d->value_0C = x;` is the right chaining order.** Retail
  stores `+0xC` first; the other order swaps them.
- **A store between two volatile accesses stays between them.** Stage 10's
  `field_32 = 0` sits inside the `|= 0x10000` read-modify-write in retail, and
  writing it there in the source is what puts it there in the build.

### Source

```c
#include "../types.h"
#include "../psyq/libgte.h"
#include "../psyq/libgpu.h"

#include "file_transfer.h"

extern volatile u32 D_8009B0F4 __attribute__((section(".data")));
extern u8 *D_8009B118 __attribute__((section(".data")));
extern u8 *D_80010000 __attribute__((section(".data")));
extern u8 *D_800101DC __attribute__((section(".data")));
extern u8 D_800E9D70[100];
#define D_800E9D70 (*(RECT *)D_800E9D70)
extern u8 D_801A8000[];
extern u8 D_801A9800[];
extern u16 gDuel_awEquipTable[];
extern u16 gDuel_aFusionTable[];
extern u16 gDuel_awRitualData[];

void Duel_LoadPackageStage(FileTransferDescriptor *d, s32 stage)
{
    u32 flags;

    switch (stage) {
    case 0:
        d->counter = 0x300;
        d->field_32 = 0x100;
        d->field_04 = 0x40;
        d->field_06 = 0x10;
        D_8009B0F4 &= 0xFFDDFFFF;
        D_8009B0F4 |= 0x10000;
        d->done = 2;
        d->mode = 0x20000;
        d->value_08 = (u32)D_8009B118;
        d->value_0C = (u32)(D_8009B118 + 0x800);
        break;
    case 1:
        d->mode = 0x2000;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->value_08 = d->value_0C = (u32)D_8009B118;
        d->done = 1;
        break;
    case 2:
        D_800E9D70.x = 0x100;
        D_800E9D70.y = 0xF0;
        D_800E9D70.w = 0x100;
        D_800E9D70.h = 0x10;
        LoadImage2(&D_800E9D70, (u32 *)D_8009B118);
        d->value_08 = d->value_0C = (u32)gDuel_awEquipTable;
        d->mode = 0x2800;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->done = 1;
        break;
    case 3:
        d->value_08 = d->value_0C = (u32)gDuel_aFusionTable;
        d->mode = 0x10000;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->done = 1;
        break;
    case 4:
        d->value_08 = d->value_0C = (u32)gDuel_awRitualData;
        d->mode = 0x800;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->done = 1;
        break;
    case 5:
        d->mode = 0x1000;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->value_08 = d->value_0C = (u32)D_8009B118;
        d->done = 1;
        break;
    case 6:
        D_800E9D70.x = 0;
        D_800E9D70.y = 0xF0;
        D_800E9D70.w = 0x100;
        D_800E9D70.h = 8;
        LoadImage2(&D_800E9D70, (u32 *)D_8009B118);
        d->counter = 0x200;
        d->field_32 = 0x100;
        d->field_04 = 0x40;
        flags = D_8009B0F4 & 0xFFDDFFFF;
        goto image_stage;
    case 7:
        d->mode = 0x16000;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->value_08 = d->value_0C = (u32)D_800101DC;
        d->done = 1;
        break;
    case 8:
        d->value_08 = d->value_0C = (u32)D_801A8000;
        d->mode = 0x1800;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->done = 1;
        break;
    case 9:
        d->value_08 = d->value_0C = (u32)D_801A9800;
        d->mode = 0x1800;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->done = 1;
        break;
    case 10:
        d->counter = 0x340;
        d->field_04 = 0x40;
        d->field_06 = 0x10;
        D_8009B0F4 &= 0xFFDDFFFF;
        flags = D_8009B0F4;
        d->field_32 = 0;
        D_8009B0F4 = flags | 0x10000;
        d->done = 2;
        d->mode = 0x4000;
        d->value_08 = (u32)D_8009B118;
        d->value_0C = (u32)(D_8009B118 + 0x800);
        break;
    case 11:
        d->mode = 0x2800;
        D_8009B0F4 &= 0xFFDCFFFF;
        d->value_08 = d->value_0C = (u32)D_80010000;
        d->done = 1;
        break;
    case 12:
        d->counter = 0x280;
        d->field_32 = 0x100;
        d->field_04 = 0x40;
        flags = D_8009B0F4 & 0xFFDDFFFF;
    image_stage:
        D_8009B0F4 = flags;
        d->mode = 0x10000;
        D_8009B0F4 |= 0x10000;
        d->done = 2;
        d->field_06 = 0x10;
        d->value_08 = (u32)D_8009B118;
        d->value_0C = (u32)(D_8009B118 + 0x800);
        break;
    }
}
```

### file_transfer.h change

```c
typedef struct {
    u8 pad_00[0x4];
    u16 field_04;
    u16 field_06;
    u32 value_08;
    u32 value_0C;
    u8 pad_10[0xC];
    u32 mode;
    u8 pad_20[0xC];
    u32 status_flags;
    u16 counter;
    u16 field_32;
    u8 pad_34[0x4];
    void *callback_data;
    u32 position;
    u32 result;
    u8 pad_44[0x2];
    u8 done;
} FileTransferDescriptor;
```
