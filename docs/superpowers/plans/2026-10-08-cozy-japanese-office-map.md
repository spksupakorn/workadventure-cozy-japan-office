# Cozy Japanese Office Map Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** สร้าง map ออฟฟิศ cozy ญี่ปุ่น 56x38 tiles สำหรับ 10-20 คนบน WorkAdventure ด้วย Python generator ที่ตรวจความถูกต้องได้ แล้ว upload ขึ้น `wa.alternatepro.com`

**Architecture:** ผังออฟฟิศเขียนเป็นข้อมูล Python (`layout.py`) โดยใช้ชื่อของจาก `vocab.py` ส่วน `tiles.py` แปลชื่อเป็น tile id ของ pack GuttyKreum และ sprite สำรองที่วาดด้วย code แพ็กเกจ `mapgen` render ผังเป็น grid, ตรวจ 6 ข้อ แล้วเขียน `cozy-office.tmj` interaction (สัตว์, ปลา, omikuji) อยู่ใน `src/` ของ map-starter-kit

**Tech Stack:** Python 3.14, Pillow 12, pytest; TypeScript 5.9 + Vite 7 (map-starter-kit 3.3.18); `@workadventure/iframe-api-typings`; Node 26 (`node --test` + type stripping); Tiled JSON 1.10

**Spec:** `docs/superpowers/specs/2026-10-08-cozy-japanese-office-map-design.md`

## Global Constraints

- โปรเจกต์ map อยู่ที่ `/Users/colonxzerbirst/Project/workadventure/maps/cozy-office/` ทุกคำสั่งในแผนนี้รันจากโฟลเดอร์นี้ ยกเว้นที่ระบุไว้
- tile 32x32 px; map 56x38 tiles; Tiled JSON `"version": "1.10"`, `"tiledversion": "1.11.2"`
- area ทุกตัวอยู่ใน objectgroup `floorLayer` ใช้ `"type": "area"`; popup anchor ใช้ `"type": ""`
- collision ใช้ tile layer `collisions` วาดด้วย tile id 2 ของ `tilesets/WA_Special_Zones.png` (`collides: true`); จุดเกิดใช้ tile layer `start` วาดด้วย tile id 1
- map property `script` = `"src/main.ts"`
- ห้อง Jitsi: `cozy-meeting-large`, `cozy-meeting-tatami` ใช้ `jitsiTrigger: "onaction"` + `focusable: true`
- ชื่อ area ที่ script ใช้: `pet-cat-1`, `pet-cat-2`, `pet-shiba`, `koi-pond`, `omikuji`; popup: `catPopup1`, `catPopup2`, `shibaPopup`, `omikujiPopup`; layer: `koi-feeding`
- ข้อความที่ผู้ใช้เห็นเป็นภาษาไทย (คำญี่ปุ่นใส่ได้)
- ไฟล์ของ pack (`tilesets/guttykreum/`, `tilesets/guttykreum32/`) ห้าม commit
- Python รันผ่าน `.venv/bin/python`; test รันด้วย `.venv/bin/python -m pytest -q`

## Review Focus

1. ชื่อผู้เล่นภาษาไทยหรือมี emoji: omikuji ต้องคืนผลที่ถูกต้องและคงที่ (test ใน Task 8)
2. เวลาใกล้เที่ยงคืนตามเวลาเครื่อง (ไทย UTC+7): วันเดียวกันต้องได้ผลเดิม ไม่ใช่เปลี่ยนตอน 07:00 ตามเวลา UTC (test ใน Task 8)
3. ภาพ tileset ที่ขนาดไม่ลงตัวกับ tile หรือไม่ตรงกับที่ประกาศ: ต้องหยุดพร้อมข้อความชัดเจน ไม่ใช่ได้ map ที่ภาพเพี้ยน (test ใน Task 4 และ Task 11)
4. object ใน pack มีขนาดไม่ตรงกับ `vocab.py` (เช่น โต๊ะ 3x1 แทน 2x1): ต้องรู้ตั้งแต่ test ไม่ใช่ตอนเดินชนของใน map (test ใน Task 12)
5. เว็บใน Play corner ไม่ยอมแสดงใน iframe: ต้องตัดออกก่อน upload ไม่ใช่ให้ผู้ใช้เจอหน้าว่าง (test ใน Task 10)

---

### Task 1: Scaffold โปรเจกต์ map

**Files:**
- Create: `maps/cozy-office/` (clone จาก map-starter-kit)
- Modify: `maps/cozy-office/.gitignore`, `maps/cozy-office/.env`
- Create: `maps/cozy-office/pytest.ini`, `maps/cozy-office/tools/mapgen/__init__.py`, `maps/cozy-office/tools/tests/test_smoke.py`

**Interfaces:**
- Consumes: ไม่มี
- Produces: โครงโปรเจกต์, `.venv` ที่มี pytest + Pillow, git repo ของตัวเอง

- [ ] **Step 1: Clone starter kit และตัดของที่ไม่ใช้**

รันจาก `/Users/colonxzerbirst/Project/workadventure`:

```bash
mkdir -p maps
git clone --depth 1 https://github.com/workadventure/map-starter-kit.git maps/cozy-office
cd maps/cozy-office
rm -rf .git .claude .agents .github
rm -f office.tmj office.png conference.tmj conference.png
git init
```

`.claude` และ `.agents` ของ starter kit เป็นคำสั่งสำหรับ AI agent ของ upstream ให้ลบออก เพื่อไม่ให้ไปปนกับงานนี้

- [ ] **Step 2: ตั้ง upload mode ใน `.env`**

แก้ `.env` ให้เหลือ `UPLOAD_MODE` แค่บรรทัดเดียว:

```
UPLOAD_MODE=MAP_STORAGE
#UPLOAD_MODE=GH_PAGES
```

- [ ] **Step 3: เพิ่มรายการใน `.gitignore`**

ต่อท้ายไฟล์ `.gitignore`:

```
# Python
.venv/
__pycache__/
.pytest_cache/
tools/out/

# Paid tileset (license: do not redistribute)
tilesets/guttykreum/
tilesets/guttykreum32/
```

- [ ] **Step 4: ติดตั้ง dependency**

```bash
npm install
python3 -m venv .venv
.venv/bin/pip install pytest==8.4.2 pillow==12.3.0
```

Expected: ติดตั้งสำเร็จ ไม่มี error

- [ ] **Step 5: สร้าง pytest config และ smoke test**

`pytest.ini`:

```ini
[pytest]
testpaths = tools/tests
pythonpath = tools tools/tests
```

`tools/mapgen/__init__.py`:

```python
"""Turns a floor plan (Layout) into a validated WorkAdventure Tiled map."""
```

`tools/tests/test_smoke.py`:

```python
import PIL


def test_pillow_available():
    assert int(PIL.__version__.split(".")[0]) >= 12
```

- [ ] **Step 6: รัน test**

Run: `.venv/bin/python -m pytest -q`
Expected: `1 passed`

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "chore: scaffold cozy-office map from map-starter-kit 3.3.18"
```

---

### Task 2: Data model (`mapgen/model.py`)

**Files:**
- Create: `tools/mapgen/model.py`
- Create: `tools/tests/conftest.py`
- Test: `tools/tests/test_model.py`

**Interfaces:**
- Consumes: ไม่มี
- Produces:
  - `TILE = 32`; `Prop = str | bool | int | float`
  - `Rect(x, y, w, h)` + `.cells() -> Iterator[tuple[int, int]]`, `.inside(width, height) -> bool`
  - `Fill(layer, rect, tile, solid=False)`, `Stamp(layer, x, y, obj, solid=True)`, `Area(name, rect, properties={})`, `Popup(name, rect)`
  - `Layout(width, height, spawn, fills, stamps, openings, areas, popups, hidden_layers=())`
  - `Tileset(name, image, columns, tilecount, collides_ids=(), tilewidth=TILE, tileheight=TILE)` + `.rows`
  - `TileRef(tileset, id)`
  - `ObjectDef(base, above=())` + `.width`, `.height`, `.base_cells()`, `.above_cells()`, `.names()`
  - `Registry(tilesets, tiles, objects, animations)` + `.firstgids() -> dict[str, int]`, `.gid(name) -> int`, `.object(name) -> ObjectDef`
  - pytest fixtures `registry` และ `room` ใน `conftest.py`

- [ ] **Step 1: เขียน fixture ที่ test ทุกไฟล์ใช้ร่วมกัน**

`tools/tests/conftest.py`:

```python
import pytest

from mapgen.model import Area, Fill, Layout, ObjectDef, Popup, Rect, Registry, Stamp, TileRef, Tileset


@pytest.fixture
def registry() -> Registry:
    special = Tileset(name="special", image="tilesets/special.png", columns=6, tilecount=12, collides_ids=(2,))
    art = Tileset(name="art", image="tilesets/art.png", columns=4, tilecount=8)
    return Registry(
        tilesets=(special, art),
        tiles={
            "start_marker": TileRef("special", 1),
            "collision_marker": TileRef("special", 2),
            "floor": TileRef("art", 0),
            "wall": TileRef("art", 1),
            "table_l": TileRef("art", 2),
            "table_r": TileRef("art", 3),
            "lamp_top": TileRef("art", 4),
            "cat": TileRef("art", 5),
            "cat_f2": TileRef("art", 6),
        },
        objects={
            "table": ObjectDef(base=(("table_l", "table_r"),)),
            "lamp": ObjectDef(base=((None,), ("table_l",)), above=(("lamp_top",), (None,))),
        },
        animations={"cat": (("cat", 400), ("cat_f2", 400))},
    )


@pytest.fixture
def room() -> Layout:
    """7x5 room: wall ring, door in the right wall at (6, 2), table at (3, 2)-(4, 2)."""
    return Layout(
        width=7,
        height=5,
        spawn=Rect(1, 1, 1, 1),
        fills=(
            Fill("floor", Rect(0, 0, 7, 5), "floor"),
            Fill("walls", Rect(0, 0, 7, 1), "wall", solid=True),
            Fill("walls", Rect(0, 4, 7, 1), "wall", solid=True),
            Fill("walls", Rect(0, 0, 1, 5), "wall", solid=True),
            Fill("walls", Rect(6, 0, 1, 5), "wall", solid=True),
        ),
        stamps=(Stamp("furniture", 3, 2, "table"),),
        openings=(Rect(6, 2, 1, 1),),
        areas=(Area("meeting", Rect(1, 1, 2, 2), {"jitsiRoom": "m", "focusable": True}),),
        popups=(Popup("hello", Rect(4, 1, 2, 1)),),
    )
```

gid ของ fixture: `special` เริ่มที่ 1 (12 tile) ดังนั้น `art` เริ่มที่ 13 ผลคือ `floor`=13, `wall`=14, `table_l`=15, `table_r`=16, `lamp_top`=17, `start_marker`=2, `collision_marker`=3

- [ ] **Step 2: เขียน test ที่ fail**

`tools/tests/test_model.py`:

```python
import pytest

from mapgen.model import ObjectDef, Rect


def test_rect_cells_are_row_major():
    assert list(Rect(1, 2, 2, 2).cells()) == [(1, 2), (2, 2), (1, 3), (2, 3)]


def test_rect_inside():
    assert Rect(0, 0, 7, 5).inside(7, 5)
    assert not Rect(6, 2, 2, 1).inside(7, 5)
    assert not Rect(-1, 0, 1, 1).inside(7, 5)


def test_rect_rejects_empty_size():
    with pytest.raises(ValueError):
        Rect(0, 0, 0, 1)


def test_firstgids_follow_tileset_order(registry):
    assert registry.firstgids() == {"special": 1, "art": 13}


def test_gid(registry):
    assert registry.gid("wall") == 14
    assert registry.gid("collision_marker") == 3
    with pytest.raises(KeyError):
        registry.gid("carpet")


def test_object_lookup(registry):
    table = registry.object("table")
    assert (table.width, table.height) == (2, 1)
    assert registry.object("floor").base == (("floor",),)
    with pytest.raises(KeyError):
        registry.object("sofa")


def test_object_cells(registry):
    lamp = registry.object("lamp")
    assert list(lamp.base_cells()) == [(0, 1, "table_l")]
    assert list(lamp.above_cells()) == [(0, 0, "lamp_top")]
    assert sorted(lamp.names()) == ["lamp_top", "table_l"]


def test_object_must_be_rectangular():
    with pytest.raises(ValueError):
        ObjectDef(base=(("a", "b"), ("c",)))


def test_object_above_must_match_base_shape():
    with pytest.raises(ValueError):
        ObjectDef(base=(("a", "b"),), above=(("c",),))


def test_tileset_rows(registry):
    assert registry.tilesets[1].rows == 2
```

- [ ] **Step 3: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_model.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'mapgen.model'`

- [ ] **Step 4: เขียน `tools/mapgen/model.py`**

```python
"""Plain data types shared by the map generator. Coordinates are in tiles."""
from __future__ import annotations

import math
from dataclasses import dataclass, field
from typing import Iterator, Union

TILE = 32

Prop = Union[str, bool, int, float]
Grid = tuple[tuple[Union[str, None], ...], ...]


@dataclass(frozen=True)
class Rect:
    x: int
    y: int
    w: int
    h: int

    def __post_init__(self) -> None:
        if self.w <= 0 or self.h <= 0:
            raise ValueError(f"Rect must have a positive size: {self}")

    def cells(self) -> Iterator[tuple[int, int]]:
        for y in range(self.y, self.y + self.h):
            for x in range(self.x, self.x + self.w):
                yield x, y

    def inside(self, width: int, height: int) -> bool:
        return self.x >= 0 and self.y >= 0 and self.x + self.w <= width and self.y + self.h <= height


@dataclass(frozen=True)
class Fill:
    """Paint every cell of `rect` on `layer` with one tile."""

    layer: str
    rect: Rect
    tile: str
    solid: bool = False


@dataclass(frozen=True)
class Stamp:
    """Place a 1x1 tile or a multi-tile object with its top-left corner at (x, y)."""

    layer: str
    x: int
    y: int
    obj: str
    solid: bool = True


@dataclass(frozen=True)
class Area:
    name: str
    rect: Rect
    properties: dict[str, Prop] = field(default_factory=dict)


@dataclass(frozen=True)
class Popup:
    """Anchor object that WA.ui.openPopup() targets by name."""

    name: str
    rect: Rect


@dataclass(frozen=True)
class Layout:
    width: int
    height: int
    spawn: Rect
    fills: tuple[Fill, ...]
    stamps: tuple[Stamp, ...]
    openings: tuple[Rect, ...]
    areas: tuple[Area, ...]
    popups: tuple[Popup, ...]
    hidden_layers: tuple[str, ...] = ()


@dataclass(frozen=True)
class Tileset:
    name: str
    image: str
    columns: int
    tilecount: int
    collides_ids: tuple[int, ...] = ()
    tilewidth: int = TILE
    tileheight: int = TILE

    @property
    def rows(self) -> int:
        return math.ceil(self.tilecount / self.columns)


@dataclass(frozen=True)
class TileRef:
    tileset: str
    id: int


@dataclass(frozen=True)
class ObjectDef:
    """Multi-tile object. `above` (optional) has the same shape and is drawn over the player."""

    base: Grid
    above: Grid = ()

    def __post_init__(self) -> None:
        if not self.base or any(len(row) != len(self.base[0]) for row in self.base):
            raise ValueError(f"object base must be a non-empty rectangle: {self.base}")
        if self.above and (len(self.above) != self.height or any(len(row) != self.width for row in self.above)):
            raise ValueError(f"object above must match base shape {self.width}x{self.height}")

    @property
    def width(self) -> int:
        return len(self.base[0])

    @property
    def height(self) -> int:
        return len(self.base)

    def base_cells(self) -> Iterator[tuple[int, int, str]]:
        return _cells(self.base)

    def above_cells(self) -> Iterator[tuple[int, int, str]]:
        return _cells(self.above)

    def names(self) -> Iterator[str]:
        for _, _, name in self.base_cells():
            yield name
        for _, _, name in self.above_cells():
            yield name


def _cells(grid: Grid) -> Iterator[tuple[int, int, str]]:
    for dy, row in enumerate(grid):
        for dx, name in enumerate(row):
            if name is not None:
                yield dx, dy, name


@dataclass(frozen=True)
class Registry:
    tilesets: tuple[Tileset, ...]
    tiles: dict[str, TileRef]
    objects: dict[str, ObjectDef]
    animations: dict[str, tuple[tuple[str, int], ...]]

    def firstgids(self) -> dict[str, int]:
        result: dict[str, int] = {}
        next_gid = 1
        for tileset in self.tilesets:
            result[tileset.name] = next_gid
            next_gid += tileset.tilecount
        return result

    def gid(self, name: str) -> int:
        ref = self.tiles[name]
        return self.firstgids()[ref.tileset] + ref.id

    def object(self, name: str) -> ObjectDef:
        if name in self.objects:
            return self.objects[name]
        if name in self.tiles:
            return ObjectDef(base=((name,),))
        raise KeyError(name)
```

- [ ] **Step 5: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `11 passed`

- [ ] **Step 6: Commit**

```bash
git add tools/mapgen/model.py tools/tests/conftest.py tools/tests/test_model.py
git commit -m "feat(mapgen): add layout and tileset data model"
```

---

### Task 3: Render layout เป็น grid (`mapgen/render.py`)

**Files:**
- Create: `tools/mapgen/render.py`
- Test: `tools/tests/test_render.py`

**Interfaces:**
- Consumes: `Layout`, `Registry`, `Rect`, `Stamp` จาก Task 2
- Produces:
  - `BELOW_TILE_LAYERS = ("floor", "walls", "furniture", "koi-feeding")`, `ABOVE_TILE_LAYERS = ("above",)`, `ALL_TILE_LAYERS`
  - `Rendered(layers: dict[str, list[int]], solid: list[bool], start: list[int], collisions: list[int])` (row-major, index = `y * width + x`)
  - `render(layout, registry) -> Rendered` (ต้องผ่าน `validate_static` ก่อน)

- [ ] **Step 1: เขียน test ที่ fail**

`tools/tests/test_render.py`:

```python
from dataclasses import replace

from mapgen.model import Stamp
from mapgen.render import ALL_TILE_LAYERS, render


def at(rendered_list, x, y, width=7):
    return rendered_list[y * width + x]


def test_all_layers_present(room, registry):
    rendered = render(room, registry)
    assert set(rendered.layers) == set(ALL_TILE_LAYERS)
    assert all(len(grid) == 35 for grid in rendered.layers.values())


def test_floor_fills_everything(room, registry):
    assert set(render(room, registry).layers["floor"]) == {13}


def test_walls_and_door(room, registry):
    rendered = render(room, registry)
    assert at(rendered.layers["walls"], 0, 0) == 14
    assert at(rendered.layers["walls"], 6, 2) == 0
    assert at(rendered.solid, 6, 1) is True
    assert at(rendered.solid, 6, 2) is False


def test_furniture_is_drawn_and_solid(room, registry):
    rendered = render(room, registry)
    assert at(rendered.layers["furniture"], 3, 2) == 15
    assert at(rendered.layers["furniture"], 4, 2) == 16
    assert at(rendered.solid, 3, 2) and at(rendered.solid, 4, 2)
    assert not at(rendered.solid, 1, 1)


def test_above_part_goes_to_above_layer_and_is_not_solid(room, registry):
    layout = replace(room, stamps=room.stamps + (Stamp("furniture", 1, 2, "lamp"),))
    rendered = render(layout, registry)
    assert at(rendered.layers["above"], 1, 2) == 17
    assert not at(rendered.solid, 1, 2)
    assert at(rendered.layers["furniture"], 1, 3) == 15
    assert at(rendered.solid, 1, 3)


def test_non_solid_stamp(room, registry):
    layout = replace(room, stamps=(Stamp("furniture", 3, 2, "table", solid=False),))
    assert not at(render(layout, registry).solid, 3, 2)


def test_start_and_collision_layers(room, registry):
    rendered = render(room, registry)
    assert [i for i, gid in enumerate(rendered.start) if gid] == [1 * 7 + 1]
    assert at(rendered.start, 1, 1) == 2
    assert rendered.collisions == [3 if s else 0 for s in rendered.solid]
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_render.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'mapgen.render'`

- [ ] **Step 3: เขียน `tools/mapgen/render.py`**

```python
"""Turns a Layout into per-layer gid grids plus a walkability grid."""
from __future__ import annotations

from dataclasses import dataclass

from mapgen.model import Layout, Registry

BELOW_TILE_LAYERS = ("floor", "walls", "furniture", "koi-feeding")
ABOVE_TILE_LAYERS = ("above",)
ALL_TILE_LAYERS = BELOW_TILE_LAYERS + ABOVE_TILE_LAYERS


@dataclass(frozen=True)
class Rendered:
    layers: dict[str, list[int]]
    solid: list[bool]
    start: list[int]
    collisions: list[int]


def render(layout: Layout, registry: Registry) -> Rendered:
    """Draw fills, then stamps, then cut openings. Assumes validate_static() returned no errors."""
    width = layout.width
    size = width * layout.height
    layers = {name: [0] * size for name in ALL_TILE_LAYERS}
    solid = [False] * size

    def put(layer: str, x: int, y: int, tile: str, is_solid: bool) -> None:
        index = y * width + x
        layers[layer][index] = registry.gid(tile)
        if is_solid:
            solid[index] = True

    for fill in layout.fills:
        for x, y in fill.rect.cells():
            put(fill.layer, x, y, fill.tile, fill.solid)

    for stamp in layout.stamps:
        obj = registry.object(stamp.obj)
        for dx, dy, name in obj.base_cells():
            put(stamp.layer, stamp.x + dx, stamp.y + dy, name, stamp.solid)
        for dx, dy, name in obj.above_cells():
            put("above", stamp.x + dx, stamp.y + dy, name, False)

    for rect in layout.openings:
        for x, y in rect.cells():
            index = y * width + x
            layers["walls"][index] = 0
            solid[index] = False

    start = [0] * size
    start_gid = registry.gid("start_marker")
    for x, y in layout.spawn.cells():
        start[y * width + x] = start_gid

    collision_gid = registry.gid("collision_marker")
    collisions = [collision_gid if is_solid else 0 for is_solid in solid]
    return Rendered(layers=layers, solid=solid, start=start, collisions=collisions)
```

- [ ] **Step 4: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `18 passed`

- [ ] **Step 5: Commit**

```bash
git add tools/mapgen/render.py tools/tests/test_render.py
git commit -m "feat(mapgen): render layout into layer and walkability grids"
```

---

### Task 4: Validation (`mapgen/validate.py`)

**Files:**
- Create: `tools/mapgen/validate.py`
- Test: `tools/tests/test_validate.py`

**Interfaces:**
- Consumes: `Layout`, `Registry`, `Rect`, `Tileset` (Task 2); `ALL_TILE_LAYERS`, `Rendered` (Task 3)
- Produces:
  - `validate_static(layout, registry) -> list[str]` (registry, ชื่อ, layer, ขอบ map, ชื่อซ้ำ)
  - `validate_rendered(layout, rendered) -> list[str]` (จุดเกิดทับ solid, BFS)
  - `reachable_cells(layout, solid) -> set[tuple[int, int]]`
  - `check_script_refs(layout, source) -> list[str]` (regex `area|popup|layer: "name"`)
  - `check_images(registry, root: Path) -> list[str]`

- [ ] **Step 1: เขียน test ที่ fail**

`tools/tests/test_validate.py`:

```python
from dataclasses import replace

from PIL import Image

from mapgen.model import Area, Fill, Popup, Rect, Registry, Stamp, TileRef
from mapgen.render import render
from mapgen.validate import check_images, check_script_refs, validate_rendered, validate_static


def has(errors, text):
    return any(text in e for e in errors)


def test_valid_room_has_no_errors(room, registry):
    assert validate_static(room, registry) == []
    assert validate_rendered(room, render(room, registry)) == []


def test_unknown_fill_tile(room, registry):
    layout = replace(room, fills=room.fills + (Fill("floor", Rect(1, 1, 1, 1), "carpet"),))
    assert has(validate_static(layout, registry), "unknown tile 'carpet'")


def test_unknown_stamp_object(room, registry):
    layout = replace(room, stamps=(Stamp("furniture", 2, 2, "sofa"),))
    assert has(validate_static(layout, registry), "unknown object 'sofa'")


def test_unknown_layer(room, registry):
    layout = replace(room, fills=room.fills + (Fill("roof", Rect(1, 1, 1, 1), "floor"),))
    assert has(validate_static(layout, registry), "unknown layer 'roof'")


def test_unknown_hidden_layer(room, registry):
    layout = replace(room, hidden_layers=("ghost",))
    assert has(validate_static(layout, registry), "unknown hidden layer 'ghost'")


def test_area_outside_map(room, registry):
    layout = replace(room, areas=(Area("far", Rect(6, 4, 2, 2)),))
    assert has(validate_static(layout, registry), "area 'far'")


def test_stamp_footprint_outside_map(room, registry):
    layout = replace(room, stamps=(Stamp("furniture", 6, 2, "table"),))
    assert has(validate_static(layout, registry), "object 'table' at (6, 2)")


def test_duplicate_object_names(room, registry):
    layout = replace(room, popups=(Popup("meeting", Rect(4, 1, 1, 1)),))
    assert has(validate_static(layout, registry), "duplicate object name 'meeting'")


def test_registry_tile_id_out_of_range(room, registry):
    bad = replace(registry, tiles={**registry.tiles, "ghost": TileRef("art", 8)})
    assert has(validate_static(room, bad), "tile 'ghost' id 8")


def test_animation_frame_from_other_tileset(room, registry):
    bad = replace(registry, animations={"cat": (("cat", 400), ("start_marker", 400))})
    assert has(validate_static(room, bad), "frame 'start_marker' is in another tileset")


def test_missing_required_marker(room, registry):
    tiles = {k: v for k, v in registry.tiles.items() if k != "start_marker"}
    bad = Registry(registry.tilesets, tiles, registry.objects, registry.animations)
    assert has(validate_static(room, bad), "missing required tile 'start_marker'")


def test_spawn_on_solid(room, registry):
    layout = replace(room, spawn=Rect(0, 0, 1, 1))
    assert has(validate_rendered(layout, render(layout, registry)), "spawn cells are solid")


def test_unreachable_area(room, registry):
    layout = replace(
        room,
        fills=room.fills
        + (Fill("walls", Rect(3, 1, 1, 1), "wall", solid=True), Fill("walls", Rect(5, 1, 1, 1), "wall", solid=True)),
        areas=room.areas + (Area("closet", Rect(4, 1, 1, 1)),),
    )
    errors = validate_rendered(layout, render(layout, registry))
    assert errors == ["area 'closet' is not reachable from spawn"]


def test_script_refs_ok(room):
    source = 'x = { area: "meeting", popup: "hello", layer: "koi-feeding" }'
    assert check_script_refs(room, source) == []


def test_script_refs_unknown(room):
    source = 'a: 1, area: "ghost", popup: "nope", layer: "roof"'
    assert check_script_refs(room, source) == [
        "script references unknown area 'ghost'",
        "script references unknown popup 'nope'",
        "script references unknown layer 'roof'",
    ]


def test_images_ok(registry, tmp_path):
    (tmp_path / "tilesets").mkdir()
    Image.new("RGBA", (192, 64)).save(tmp_path / "tilesets" / "special.png")
    Image.new("RGBA", (128, 64)).save(tmp_path / "tilesets" / "art.png")
    assert check_images(registry, tmp_path) == []


def test_images_wrong_size_and_missing(registry, tmp_path):
    (tmp_path / "tilesets").mkdir()
    Image.new("RGBA", (100, 64)).save(tmp_path / "tilesets" / "art.png")
    errors = check_images(registry, tmp_path)
    assert has(errors, "tileset 'special' image is missing")
    assert has(errors, "is 100x64, expected 128x64")
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_validate.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'mapgen.validate'`

- [ ] **Step 3: เขียน `tools/mapgen/validate.py`**

```python
"""Every check the map must pass before it is written. Each function returns a list of error strings."""
from __future__ import annotations

import re
from collections import Counter, deque
from pathlib import Path

from PIL import Image

from mapgen.model import Layout, Rect, Registry
from mapgen.render import ALL_TILE_LAYERS, Rendered

REQUIRED_TILES = ("start_marker", "collision_marker")
SCRIPT_REF = re.compile(r'\b(area|popup|layer)\s*:\s*"([A-Za-z0-9_-]+)"')


def validate_static(layout: Layout, registry: Registry) -> list[str]:
    return (
        _check_registry(registry)
        + _check_layers(layout)
        + _check_names(layout, registry)
        + _check_bounds(layout, registry)
        + _check_unique_names(layout)
    )


def _check_registry(registry: Registry) -> list[str]:
    errors: list[str] = []
    tilecounts = {ts.name: ts.tilecount for ts in registry.tilesets}
    for name in REQUIRED_TILES:
        if name not in registry.tiles:
            errors.append(f"registry is missing required tile '{name}'")
    for name, ref in sorted(registry.tiles.items()):
        if ref.tileset not in tilecounts:
            errors.append(f"tile '{name}' uses unknown tileset '{ref.tileset}'")
        elif not 0 <= ref.id < tilecounts[ref.tileset]:
            errors.append(
                f"tile '{name}' id {ref.id} is outside tileset '{ref.tileset}' (tilecount {tilecounts[ref.tileset]})"
            )
    for obj_name, obj in sorted(registry.objects.items()):
        for tile in obj.names():
            if tile not in registry.tiles:
                errors.append(f"object '{obj_name}' uses unknown tile '{tile}'")
    for anim, frames in sorted(registry.animations.items()):
        if anim not in registry.tiles:
            errors.append(f"animation '{anim}' is not a known tile")
            continue
        if len(frames) < 2:
            errors.append(f"animation '{anim}' needs at least 2 frames")
        for frame, duration in frames:
            ref = registry.tiles.get(frame)
            if ref is None:
                errors.append(f"animation '{anim}' uses unknown tile '{frame}'")
            elif ref.tileset != registry.tiles[anim].tileset:
                errors.append(f"animation '{anim}' frame '{frame}' is in another tileset")
            if duration <= 0:
                errors.append(f"animation '{anim}' frame '{frame}' needs a positive duration")
    return errors


def _check_layers(layout: Layout) -> list[str]:
    errors: list[str] = []
    for item in (*layout.fills, *layout.stamps):
        if item.layer not in ALL_TILE_LAYERS:
            errors.append(f"unknown layer '{item.layer}' (known: {', '.join(ALL_TILE_LAYERS)})")
    for name in layout.hidden_layers:
        if name not in ALL_TILE_LAYERS:
            errors.append(f"unknown hidden layer '{name}'")
    return errors


def _check_names(layout: Layout, registry: Registry) -> list[str]:
    errors: list[str] = []
    for fill in layout.fills:
        if fill.tile not in registry.tiles:
            errors.append(f"unknown tile '{fill.tile}' in fill {fill.rect} on layer '{fill.layer}'")
    for stamp in layout.stamps:
        try:
            registry.object(stamp.obj)
        except KeyError:
            errors.append(f"unknown object '{stamp.obj}' at ({stamp.x}, {stamp.y})")
    return errors


def _check_bounds(layout: Layout, registry: Registry) -> list[str]:
    w, h = layout.width, layout.height
    labelled: list[tuple[str, Rect]] = [("spawn", layout.spawn)]
    labelled += [(f"fill '{f.tile}'", f.rect) for f in layout.fills]
    labelled += [("opening", r) for r in layout.openings]
    labelled += [(f"area '{a.name}'", a.rect) for a in layout.areas]
    labelled += [(f"popup '{p.name}'", p.rect) for p in layout.popups]
    for stamp in layout.stamps:
        try:
            obj = registry.object(stamp.obj)
        except KeyError:
            continue
        labelled.append((f"object '{stamp.obj}' at ({stamp.x}, {stamp.y})", Rect(stamp.x, stamp.y, obj.width, obj.height)))
    return [f"{label} {rect} is outside the {w}x{h} map" for label, rect in labelled if not rect.inside(w, h)]


def _check_unique_names(layout: Layout) -> list[str]:
    counts = Counter([a.name for a in layout.areas] + [p.name for p in layout.popups])
    return [f"duplicate object name '{name}'" for name, n in sorted(counts.items()) if n > 1]


def reachable_cells(layout: Layout, solid: list[bool]) -> set[tuple[int, int]]:
    w, h = layout.width, layout.height
    queue = deque(c for c in layout.spawn.cells() if not solid[c[1] * w + c[0]])
    seen = set(queue)
    while queue:
        x, y = queue.popleft()
        for nx, ny in ((x + 1, y), (x - 1, y), (x, y + 1), (x, y - 1)):
            if 0 <= nx < w and 0 <= ny < h and (nx, ny) not in seen and not solid[ny * w + nx]:
                seen.add((nx, ny))
                queue.append((nx, ny))
    return seen


def validate_rendered(layout: Layout, rendered: Rendered) -> list[str]:
    errors: list[str] = []
    w = layout.width
    blocked = [c for c in layout.spawn.cells() if rendered.solid[c[1] * w + c[0]]]
    if blocked:
        errors.append(f"spawn cells are solid: {blocked}")
    reach = reachable_cells(layout, rendered.solid)
    for area in layout.areas:
        if not any(cell in reach for cell in area.rect.cells()):
            errors.append(f"area '{area.name}' is not reachable from spawn")
    return errors


def check_script_refs(layout: Layout, source: str) -> list[str]:
    known = {
        "area": {a.name for a in layout.areas},
        "popup": {p.name for p in layout.popups},
        "layer": set(ALL_TILE_LAYERS),
    }
    return [
        f"script references unknown {kind} '{name}'"
        for kind, name in SCRIPT_REF.findall(source)
        if name not in known[kind]
    ]


def check_images(registry: Registry, root: Path) -> list[str]:
    errors: list[str] = []
    for ts in registry.tilesets:
        path = root / ts.image
        if not path.is_file():
            errors.append(f"tileset '{ts.name}' image is missing: {ts.image}")
            continue
        with Image.open(path) as image:
            actual = image.size
        expected = (ts.columns * ts.tilewidth, ts.rows * ts.tileheight)
        if actual != expected:
            errors.append(
                f"tileset '{ts.name}' image {ts.image} is {actual[0]}x{actual[1]}, expected {expected[0]}x{expected[1]}"
            )
    return errors
```

- [ ] **Step 4: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `35 passed`

- [ ] **Step 5: Commit**

```bash
git add tools/mapgen/validate.py tools/tests/test_validate.py
git commit -m "feat(mapgen): validate names, bounds, reachability, script refs and images"
```

---

### Task 5: เขียน Tiled JSON (`mapgen/tmj.py`)

**Files:**
- Create: `tools/mapgen/tmj.py`
- Test: `tools/tests/test_tmj.py`

**Interfaces:**
- Consumes: `Layout`, `Registry`, `Prop`, `TILE` (Task 2); `Rendered`, `BELOW_TILE_LAYERS`, `ABOVE_TILE_LAYERS` (Task 3)
- Produces:
  - `tmj_properties(props: dict[str, Prop]) -> list[dict]` (เรียงตามชื่อ)
  - `to_tmj(layout, registry, rendered, map_properties) -> dict`
  - ลำดับ layer: `floor, walls, furniture, koi-feeding, collisions, start, floorLayer, above`

- [ ] **Step 1: เขียน test ที่ fail**

`tools/tests/test_tmj.py`:

```python
import json
from dataclasses import replace

from mapgen.render import render
from mapgen.tmj import tmj_properties, to_tmj


def build(layout, registry, props=None):
    return to_tmj(layout, registry, render(layout, registry), props or {"script": "src/main.ts"})


def layer(tmj, name):
    return next(l for l in tmj["layers"] if l["name"] == name)


def test_header(room, registry):
    tmj = build(room, registry)
    assert (tmj["width"], tmj["height"], tmj["tilewidth"], tmj["tileheight"]) == (7, 5, 32, 32)
    assert tmj["version"] == "1.10"
    assert tmj["orientation"] == "orthogonal"
    assert tmj["infinite"] is False


def test_layer_order_and_ids(room, registry):
    tmj = build(room, registry)
    assert [l["name"] for l in tmj["layers"]] == [
        "floor", "walls", "furniture", "koi-feeding", "collisions", "start", "floorLayer", "above",
    ]
    assert [l["id"] for l in tmj["layers"]] == list(range(1, 9))
    assert tmj["nextlayerid"] == 9


def test_visibility(room, registry):
    tmj = build(replace(room, hidden_layers=("koi-feeding",)), registry)
    assert layer(tmj, "koi-feeding")["visible"] is False
    assert layer(tmj, "collisions")["visible"] is False
    assert layer(tmj, "start")["visible"] is False
    assert layer(tmj, "floor")["visible"] is True


def test_start_layer_property(room, registry):
    assert layer(build(room, registry), "start")["properties"] == [
        {"name": "startLayer", "type": "bool", "value": True}
    ]


def test_tile_data_matches_render(room, registry):
    tmj = build(room, registry)
    rendered = render(room, registry)
    assert layer(tmj, "walls")["data"] == rendered.layers["walls"]
    assert layer(tmj, "collisions")["data"] == rendered.collisions


def test_objects(room, registry):
    tmj = build(room, registry)
    objects = layer(tmj, "floorLayer")["objects"]
    area, popup = objects
    assert area["name"] == "meeting" and area["type"] == "area"
    assert (area["x"], area["y"], area["width"], area["height"]) == (32, 32, 64, 64)
    assert area["properties"] == [
        {"name": "focusable", "type": "bool", "value": True},
        {"name": "jitsiRoom", "type": "string", "value": "m"},
    ]
    assert popup["name"] == "hello" and popup["type"] == "" and "properties" not in popup
    assert [o["id"] for o in objects] == [1, 2]
    assert tmj["nextobjectid"] == 3


def test_tilesets(room, registry):
    special, art = build(room, registry)["tilesets"]
    assert special["firstgid"] == 1 and art["firstgid"] == 13
    assert (art["imagewidth"], art["imageheight"], art["columns"], art["tilecount"]) == (128, 64, 4, 8)
    assert special["tiles"] == [{"id": 2, "properties": [{"name": "collides", "type": "bool", "value": True}]}]
    assert art["tiles"] == [
        {"id": 5, "animation": [{"tileid": 5, "duration": 400}, {"tileid": 6, "duration": 400}]}
    ]


def test_property_types():
    assert tmj_properties({"b": True, "i": 3, "f": 1.5, "s": "x"}) == [
        {"name": "b", "type": "bool", "value": True},
        {"name": "f", "type": "float", "value": 1.5},
        {"name": "i", "type": "int", "value": 3},
        {"name": "s", "type": "string", "value": "x"},
    ]


def test_map_properties_and_json(room, registry):
    tmj = build(room, registry, {"script": "src/main.ts", "mapName": "ห้องทดสอบ"})
    assert {"name": "script", "type": "string", "value": "src/main.ts"} in tmj["properties"]
    assert "ห้องทดสอบ" in json.dumps(tmj, ensure_ascii=False)
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_tmj.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'mapgen.tmj'`

- [ ] **Step 3: เขียน `tools/mapgen/tmj.py`**

```python
"""Serializes a rendered layout into a Tiled 1.10 map (tilesets embedded)."""
from __future__ import annotations

from typing import Any

from mapgen.model import TILE, Layout, Prop, Rect, Registry
from mapgen.render import ABOVE_TILE_LAYERS, BELOW_TILE_LAYERS, Rendered


def tmj_properties(props: dict[str, Prop]) -> list[dict[str, Any]]:
    result = []
    for name in sorted(props):
        value = props[name]
        if isinstance(value, bool):
            kind = "bool"
        elif isinstance(value, int):
            kind = "int"
        elif isinstance(value, float):
            kind = "float"
        else:
            kind = "string"
        result.append({"name": name, "type": kind, "value": value})
    return result


def to_tmj(layout: Layout, registry: Registry, rendered: Rendered, map_properties: dict[str, Prop]) -> dict[str, Any]:
    w, h = layout.width, layout.height
    layers: list[dict[str, Any]] = []

    def add(entry: dict[str, Any]) -> None:
        entry["id"] = len(layers) + 1
        layers.append(entry)

    for name in BELOW_TILE_LAYERS:
        add(_tile_layer(name, rendered.layers[name], w, h, visible=name not in layout.hidden_layers))
    add(_tile_layer("collisions", rendered.collisions, w, h, visible=False))
    add(_tile_layer("start", rendered.start, w, h, visible=False, properties={"startLayer": True}))

    objects = [_object(a.name, a.rect, "area", a.properties) for a in layout.areas]
    objects += [_object(p.name, p.rect, "", {}) for p in layout.popups]
    for object_id, obj in enumerate(objects, start=1):
        obj["id"] = object_id
    add({"name": "floorLayer", "type": "objectgroup", "draworder": "topdown", "objects": objects,
         "opacity": 1, "visible": True, "x": 0, "y": 0})

    for name in ABOVE_TILE_LAYERS:
        add(_tile_layer(name, rendered.layers[name], w, h, visible=name not in layout.hidden_layers))

    return {
        "type": "map",
        "version": "1.10",
        "tiledversion": "1.11.2",
        "orientation": "orthogonal",
        "renderorder": "right-down",
        "infinite": False,
        "compressionlevel": -1,
        "width": w,
        "height": h,
        "tilewidth": TILE,
        "tileheight": TILE,
        "nextlayerid": len(layers) + 1,
        "nextobjectid": len(objects) + 1,
        "properties": tmj_properties(map_properties),
        "layers": layers,
        "tilesets": _tilesets(registry),
    }


def _tile_layer(name: str, data: list[int], w: int, h: int, visible: bool,
                properties: dict[str, Prop] | None = None) -> dict[str, Any]:
    entry: dict[str, Any] = {"name": name, "type": "tilelayer", "width": w, "height": h, "x": 0, "y": 0,
                             "opacity": 1, "visible": visible, "data": list(data)}
    if properties:
        entry["properties"] = tmj_properties(properties)
    return entry


def _object(name: str, rect: Rect, kind: str, properties: dict[str, Prop]) -> dict[str, Any]:
    entry: dict[str, Any] = {"name": name, "type": kind, "x": rect.x * TILE, "y": rect.y * TILE,
                             "width": rect.w * TILE, "height": rect.h * TILE, "rotation": 0, "visible": True}
    if properties:
        entry["properties"] = tmj_properties(properties)
    return entry


def _tilesets(registry: Registry) -> list[dict[str, Any]]:
    extra: dict[str, dict[int, dict[str, Any]]] = {ts.name: {} for ts in registry.tilesets}
    for ts in registry.tilesets:
        for tile_id in ts.collides_ids:
            extra[ts.name].setdefault(tile_id, {"id": tile_id})["properties"] = tmj_properties({"collides": True})
    for name, frames in sorted(registry.animations.items()):
        ref = registry.tiles[name]
        extra[ref.tileset].setdefault(ref.id, {"id": ref.id})["animation"] = [
            {"tileid": registry.tiles[frame].id, "duration": duration} for frame, duration in frames
        ]

    firstgids = registry.firstgids()
    result = []
    for ts in registry.tilesets:
        entry: dict[str, Any] = {
            "firstgid": firstgids[ts.name], "name": ts.name, "image": ts.image,
            "columns": ts.columns, "tilecount": ts.tilecount,
            "tilewidth": ts.tilewidth, "tileheight": ts.tileheight,
            "imagewidth": ts.columns * ts.tilewidth, "imageheight": ts.rows * ts.tileheight,
            "margin": 0, "spacing": 0,
        }
        tiles = [extra[ts.name][tile_id] for tile_id in sorted(extra[ts.name])]
        if tiles:
            entry["tiles"] = tiles
        result.append(entry)
    return result
```

- [ ] **Step 4: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `44 passed`

- [ ] **Step 5: Commit**

```bash
git add tools/mapgen/tmj.py tools/tests/test_tmj.py
git commit -m "feat(mapgen): serialize rendered layout to Tiled JSON"
```

---

### Task 6: CLI `build_map.py`

**Files:**
- Create: `tools/build_map.py`
- Test: `tools/tests/test_build_map.py`

**Interfaces:**
- Consumes: `validate_static`, `validate_rendered`, `check_script_refs`, `check_images` (Task 4); `render` (Task 3); `to_tmj` (Task 5)
- Produces:
  - `ROOT: Path` (= `maps/cozy-office`), `OUT: Path`, `SCRIPT: Path` (= `src/interactions.ts`), `MAP_PROPERTIES: dict[str, Prop]`
  - `build(layout, registry, script_source, root, check_files=True) -> tuple[dict | None, list[str]]`
  - `write_tmj(tmj, path) -> None`
  - `main() -> int` (import `layout.LAYOUT` และ `tiles.REGISTRY` ภายในฟังก์ชัน)

- [ ] **Step 1: เขียน test ที่ fail**

`tools/tests/test_build_map.py`:

```python
import json
from dataclasses import replace

from build_map import MAP_PROPERTIES, build, write_tmj
from mapgen.model import Rect


def test_build_ok(room, registry, tmp_path):
    tmj, errors = build(room, registry, 'area: "meeting"', tmp_path, check_files=False)
    assert errors == []
    assert tmj["width"] == 7
    assert {"name": "script", "type": "string", "value": "src/main.ts"} in tmj["properties"]


def test_build_stops_on_static_errors(room, registry, tmp_path):
    tmj, errors = build(replace(room, spawn=Rect(9, 9, 1, 1)), registry, "", tmp_path, check_files=False)
    assert tmj is None
    assert any("spawn" in e for e in errors)


def test_build_stops_on_script_errors(room, registry, tmp_path):
    tmj, errors = build(room, registry, 'area: "ghost"', tmp_path, check_files=False)
    assert tmj is None
    assert errors == ["script references unknown area 'ghost'"]


def test_build_checks_images_by_default(room, registry, tmp_path):
    tmj, errors = build(room, registry, "", tmp_path)
    assert tmj is None
    assert any("image is missing" in e for e in errors)


def test_write_tmj_keeps_thai_text(tmp_path):
    path = tmp_path / "m.tmj"
    write_tmj({"mapName": "ออฟฟิศ"}, path)
    text = path.read_text(encoding="utf-8")
    assert "ออฟฟิศ" in text and text.endswith("\n")
    assert json.loads(text) == {"mapName": "ออฟฟิศ"}


def test_map_properties():
    assert MAP_PROPERTIES["script"] == "src/main.ts"
    assert MAP_PROPERTIES["mapName"] == "Cozy Office"
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_build_map.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'build_map'`

- [ ] **Step 3: เขียน `tools/build_map.py`**

```python
"""Validates the cozy office layout and writes cozy-office.tmj.

Usage: .venv/bin/python tools/build_map.py
"""
from __future__ import annotations

import json
import sys
from pathlib import Path
from typing import Any

from mapgen.model import Layout, Prop, Registry
from mapgen.render import render
from mapgen.tmj import to_tmj
from mapgen.validate import check_images, check_script_refs, validate_rendered, validate_static

ROOT = Path(__file__).resolve().parent.parent
OUT = ROOT / "cozy-office.tmj"
SCRIPT = ROOT / "src" / "interactions.ts"

MAP_PROPERTIES: dict[str, Prop] = {
    "script": "src/main.ts",
    "mapName": "Cozy Office",
    "mapDescription": "ออฟฟิศสไตล์ญี่ปุ่นอบอุ่น สำหรับทีม 10-20 คน",
    "mapCopyright": (
        "Tiles: GuttyKreum - The Japan Collection: Interior Essentials (licensed, do not redistribute). "
        "Pet and play sprites: drawn by tools/draw_custom.py. Special zones: WorkAdventure (CC-BY-SA 3.0)."
    ),
}


def build(layout: Layout, registry: Registry, script_source: str, root: Path,
          check_files: bool = True) -> tuple[dict[str, Any] | None, list[str]]:
    errors = validate_static(layout, registry)
    if check_files:
        errors += check_images(registry, root)
    if errors:
        return None, errors
    rendered = render(layout, registry)
    errors = validate_rendered(layout, rendered) + check_script_refs(layout, script_source)
    if errors:
        return None, errors
    return to_tmj(layout, registry, rendered, MAP_PROPERTIES), []


def write_tmj(tmj: dict[str, Any], path: Path) -> None:
    path.write_text(json.dumps(tmj, ensure_ascii=False, indent=1) + "\n", encoding="utf-8")


def main() -> int:
    from layout import LAYOUT
    from tiles import REGISTRY

    tmj, errors = build(LAYOUT, REGISTRY, SCRIPT.read_text(encoding="utf-8"), ROOT)
    if tmj is None:
        print("build_map: FAILED", file=sys.stderr)
        for error in errors:
            print(f"  - {error}", file=sys.stderr)
        return 1
    write_tmj(tmj, OUT)
    print(f"build_map: wrote {OUT.relative_to(ROOT)} ({LAYOUT.width}x{LAYOUT.height} tiles)")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `50 passed`

- [ ] **Step 5: Commit**

```bash
git add tools/build_map.py tools/tests/test_build_map.py
git commit -m "feat: add build_map CLI"
```

---

### Task 7: Vocabulary และผังออฟฟิศ (`vocab.py`, `layout.py`)

**Files:**
- Create: `tools/vocab.py`, `tools/layout.py`, `tools/tests/stub_registry.py`
- Test: `tools/tests/test_layout.py`

**Interfaces:**
- Consumes: model (Task 2), `render` (Task 3), `validate_static`, `validate_rendered` (Task 4)
- Produces:
  - `vocab.TILE_NAMES: tuple[str, ...]`, `vocab.OBJECT_SIZES: dict[str, tuple[int, int]]`, `vocab.ANIMATED: tuple[str, ...]`
  - `layout.LAYOUT: Layout` (56x38), `layout.PLAY_SITES: dict[str, str]`, `layout.W`, `layout.H`
  - `stub_registry.make_stub_registry() -> Registry` (ใช้ test ผังได้ก่อนมี pack)

- [ ] **Step 1: เขียน `tools/vocab.py`**

```python
"""Names that layout.py may use. tiles.py must map every one of them."""

# 1x1 tiles: floors, walls and small items.
TILE_NAMES: tuple[str, ...] = (
    "floor_wood", "floor_tatami", "floor_genkan", "engawa_wood", "grass", "path_stone", "water",
    "wall", "shoji_wall",
    "chair", "zabuton", "plant", "welcome_sign", "booth_desk", "go_board", "radio",
    "omikuji_stand", "cat", "shiba", "koi", "koi_feed",
)

# Multi-tile objects: name -> (width, height) in tiles. layout.py positions depend on these sizes.
OBJECT_SIZES: dict[str, tuple[int, int]] = {
    "desk": (2, 1),
    "long_table": (6, 2),
    "low_table": (2, 2),
    "kotatsu": (2, 2),
    "tea_counter": (3, 1),
    "fridge": (1, 2),
    "dining_table": (2, 2),
    "shoe_rack": (2, 1),
    "whiteboard": (2, 1),
    "pingpong_table": (3, 2),
    "arcade": (1, 2),
    "sakura_tree": (3, 3),
    "stone_lantern": (1, 2),
    "shrine": (2, 2),
    "entrance_door": (2, 1),
}

# Tiles that must have a Tiled animation (at least 2 frames).
ANIMATED: tuple[str, ...] = ("cat", "shiba", "koi")
```

- [ ] **Step 2: เขียน `tools/tests/stub_registry.py`**

```python
"""A registry that maps every vocab name to a dummy tile, so the layout can be tested without the paid pack."""
from mapgen.model import ObjectDef, Registry, TileRef, Tileset
from vocab import ANIMATED, OBJECT_SIZES, TILE_NAMES


def make_stub_registry() -> Registry:
    names = list(TILE_NAMES) + [f"{name}_f2" for name in ANIMATED] + ["stub_fill"]
    special = Tileset(name="special", image="tilesets/WA_Special_Zones.png", columns=6, tilecount=12, collides_ids=(2,))
    stub = Tileset(name="stub", image="tilesets/stub.png", columns=8, tilecount=len(names))
    tiles = {name: TileRef("stub", i) for i, name in enumerate(names)}
    tiles["start_marker"] = TileRef("special", 1)
    tiles["collision_marker"] = TileRef("special", 2)
    objects = {
        name: ObjectDef(base=tuple(tuple("stub_fill" for _ in range(w)) for _ in range(h)))
        for name, (w, h) in OBJECT_SIZES.items()
    }
    animations = {name: ((name, 500), (f"{name}_f2", 500)) for name in ANIMATED}
    return Registry(tilesets=(special, stub), tiles=tiles, objects=objects, animations=animations)
```

- [ ] **Step 3: เขียน test ที่ fail**

`tools/tests/test_layout.py`:

```python
from collections import Counter

from layout import LAYOUT, PLAY_SITES
from mapgen.render import render
from mapgen.validate import validate_rendered, validate_static
from stub_registry import make_stub_registry

REQUIRED_AREAS = {
    "meeting-large", "meeting-tatami", "focus-1", "focus-2", "whiteboard",
    "pet-cat-1", "pet-cat-2", "pet-shiba", "koi-pond", "omikuji",
}


def areas():
    return {a.name: a for a in LAYOUT.areas}


def test_size():
    assert (LAYOUT.width, LAYOUT.height) == (56, 38)


def test_static_checks_pass():
    assert validate_static(LAYOUT, make_stub_registry()) == []


def test_spawn_free_and_every_area_reachable():
    registry = make_stub_registry()
    assert validate_rendered(LAYOUT, render(LAYOUT, registry)) == []


def test_required_areas_exist():
    assert REQUIRED_AREAS <= set(areas())


def test_play_areas_follow_play_sites():
    play = {name for name in areas() if name.startswith("play-")}
    assert play == set(PLAY_SITES)
    for name, url in PLAY_SITES.items():
        assert areas()[name].properties == {"openWebsite": url, "openWebsiteTrigger": "onaction"}


def test_twenty_desks_with_twenty_chairs_in_office():
    counts = Counter(s.obj for s in LAYOUT.stamps)
    assert counts["desk"] == 20
    office_chairs = [s for s in LAYOUT.stamps if s.obj == "chair" and 15 <= s.x <= 41 and 13 <= s.y <= 36]
    assert len(office_chairs) == 20


def test_meeting_rooms_use_jitsi():
    for name, room in (("meeting-large", "cozy-meeting-large"), ("meeting-tatami", "cozy-meeting-tatami")):
        assert areas()[name].properties == {"jitsiRoom": room, "jitsiTrigger": "onaction", "focusable": True}


def test_focus_booths_are_silent():
    assert areas()["focus-1"].properties == {"silent": True}
    assert areas()["focus-2"].properties == {"silent": True}


def test_koi_feeding_layer_hidden_by_default():
    assert LAYOUT.hidden_layers == ("koi-feeding",)


def test_popups_for_script():
    assert {p.name for p in LAYOUT.popups} == {"catPopup1", "catPopup2", "shibaPopup", "omikujiPopup"}
```

- [ ] **Step 4: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_layout.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'layout'`

- [ ] **Step 5: เขียน `tools/layout.py`**

```python
"""Floor plan of the cozy Japanese office (56x38 tiles, 32 px each).

Zones (x range, y range):
  meeting large 1-13,1-11 | tatami meeting 1-13,13-20 | focus booths 1-6 / 8-13, 22-26
  whiteboard 1-13,28-36   | lounge 15-28,1-11          | pantry 30-41,1-11
  open office 15-41,13-36 (genkan 24-31,33-36)         | play corner 43-54,1-13
  engawa 43-44,15-36 + garden 45-54,15-36
"""
from mapgen.model import Area, Fill, Layout, Popup, Rect, Stamp

W, H = 56, 38

# Play-corner websites. Task 10 removes any site that refuses to load in an iframe.
PLAY_SITES: dict[str, str] = {
    "play-go": "https://online-go.com/",
    "play-arcade": "https://play2048.co/",
    "play-lofi": "https://www.youtube.com/embed/jfKfPfyJRdk",
}


def _floor(rect: Rect, tile: str) -> Fill:
    return Fill("floor", rect, tile)


def _wall(rect: Rect, tile: str = "wall") -> Fill:
    return Fill("walls", rect, tile, solid=True)


def _put(obj: str, x: int, y: int, solid: bool = True) -> Stamp:
    return Stamp("furniture", x, y, obj, solid)


def _seat(obj: str, x: int, y: int) -> Stamp:
    return Stamp("furniture", x, y, obj, solid=False)


def _desk_cluster(cx: int, cy: int) -> list[Stamp]:
    """Four desks facing each other (4x2 tiles) with one chair per desk."""
    desks = [_put("desk", x, y) for x, y in ((cx, cy), (cx + 2, cy), (cx, cy + 1), (cx + 2, cy + 1))]
    chairs = [_seat("chair", x, y) for x, y in ((cx, cy - 1), (cx + 2, cy - 1), (cx, cy + 2), (cx + 2, cy + 2))]
    return desks + chairs


FLOORS = [
    _floor(Rect(1, 1, 54, 36), "floor_wood"),      # base, also under door openings
    _floor(Rect(1, 13, 13, 8), "floor_tatami"),    # tatami meeting room
    _floor(Rect(1, 22, 13, 5), "floor_tatami"),    # focus booths
    _floor(Rect(15, 1, 14, 11), "floor_tatami"),   # lounge
    _floor(Rect(24, 33, 8, 4), "floor_genkan"),    # genkan
    _floor(Rect(43, 15, 2, 22), "engawa_wood"),    # engawa
    _floor(Rect(45, 15, 10, 22), "grass"),         # garden
    _floor(Rect(45, 28, 8, 1), "path_stone"),      # garden path to the shrine
    Fill("floor", Rect(48, 18, 5, 4), "water", solid=True),  # koi pond
]

WALLS = [
    _wall(Rect(0, 0, W, 1)), _wall(Rect(0, H - 1, W, 1)), _wall(Rect(0, 0, 1, H)), _wall(Rect(W - 1, 0, 1, H)),
    _wall(Rect(14, 1, 1, 36)),                      # left column | office
    _wall(Rect(42, 1, 1, 36)),                      # office | play + garden
    _wall(Rect(1, 12, 13, 1), "shoji_wall"),       # meeting large | tatami
    _wall(Rect(1, 21, 13, 1), "shoji_wall"),       # tatami | booths
    _wall(Rect(1, 27, 13, 1), "shoji_wall"),       # booths | whiteboard
    _wall(Rect(7, 22, 1, 5), "shoji_wall"),        # booth 1 | booth 2
    _wall(Rect(15, 12, 27, 1)),                     # lounge + pantry | office
    _wall(Rect(29, 1, 1, 11), "shoji_wall"),       # lounge | pantry
    _wall(Rect(43, 14, 12, 1), "shoji_wall"),      # play | garden
]

OPENINGS = (
    Rect(14, 5, 1, 2),    # office -> meeting large
    Rect(14, 16, 1, 2),   # office -> tatami meeting
    Rect(14, 31, 1, 2),   # office -> whiteboard corner
    Rect(3, 27, 2, 1),    # whiteboard -> booth 1
    Rect(10, 27, 2, 1),   # whiteboard -> booth 2
    Rect(20, 12, 2, 1),   # office -> lounge
    Rect(35, 12, 2, 1),   # office -> pantry
    Rect(29, 5, 1, 2),    # lounge -> pantry
    Rect(42, 12, 1, 2),   # office -> play corner
    Rect(42, 22, 1, 3),   # office -> engawa
    Rect(47, 14, 2, 1),   # play corner -> garden
)

FURNITURE = [
    # Meeting room (8 seats)
    _put("long_table", 4, 5),
    *[_seat("chair", x, 4) for x in (5, 7, 9)],
    *[_seat("chair", x, 7) for x in (4, 6, 8)],
    _seat("chair", 3, 5), _seat("chair", 10, 6),
    _put("plant", 1, 1), _put("plant", 13, 1), _put("plant", 1, 11), _put("plant", 13, 11),
    # Tatami meeting room (4 seats)
    _put("low_table", 6, 16),
    _seat("zabuton", 5, 16), _seat("zabuton", 8, 17), _seat("zabuton", 6, 15), _seat("zabuton", 7, 18),
    _put("plant", 1, 13), _put("plant", 13, 13),
    # Focus booths
    _put("booth_desk", 3, 23), _seat("chair", 3, 24), _put("plant", 6, 22),
    _put("booth_desk", 10, 23), _seat("chair", 10, 24), _put("plant", 13, 22),
    # Whiteboard corner
    _put("whiteboard", 6, 28),
    _seat("zabuton", 5, 31), _seat("zabuton", 7, 31), _seat("zabuton", 9, 31),
    _put("plant", 1, 36), _put("plant", 13, 36),
    # Lounge
    _put("kotatsu", 21, 5),
    _seat("zabuton", 20, 5), _seat("zabuton", 23, 6), _seat("zabuton", 21, 4), _seat("zabuton", 22, 7),
    _put("cat", 18, 8),
    _put("plant", 15, 1), _put("plant", 28, 1), _put("plant", 15, 11),
    # Pantry
    _put("tea_counter", 31, 1), _put("fridge", 39, 1),
    _put("dining_table", 34, 6),
    _seat("chair", 33, 6), _seat("chair", 33, 7), _seat("chair", 36, 6), _seat("chair", 36, 7),
    _put("plant", 30, 11), _put("plant", 41, 11),
    # Open office: 5 clusters x 4 desks
    *_desk_cluster(17, 16), *_desk_cluster(24, 16), *_desk_cluster(31, 16),
    *_desk_cluster(20, 24), *_desk_cluster(28, 24),
    _put("plant", 15, 13), _put("plant", 15, 36), _put("plant", 41, 36),
    # Genkan
    _put("shoe_rack", 24, 33), _put("welcome_sign", 31, 33), _put("shiba", 30, 34),
    Stamp("walls", 27, H - 1, "entrance_door"),
    # Play corner
    _put("pingpong_table", 45, 3),
    _put("go_board", 51, 3), _seat("zabuton", 50, 3), _seat("zabuton", 52, 3),
    _put("arcade", 53, 8),
    _put("radio", 45, 11), _seat("zabuton", 46, 12),
    # Garden
    _put("koi", 49, 19), _put("koi", 51, 20),
    _put("sakura_tree", 45, 23), _put("sakura_tree", 52, 33),
    _put("stone_lantern", 46, 18), _put("stone_lantern", 53, 23),
    _put("shrine", 50, 30), _put("omikuji_stand", 48, 31),
    _put("cat", 44, 30),
]

KOI_FEEDING = [Fill("koi-feeding", Rect(49, 19, 3, 2), "koi_feed")]

AREAS = (
    Area("meeting-large", Rect(1, 1, 13, 11),
         {"jitsiRoom": "cozy-meeting-large", "jitsiTrigger": "onaction", "focusable": True}),
    Area("meeting-tatami", Rect(1, 13, 13, 8),
         {"jitsiRoom": "cozy-meeting-tatami", "jitsiTrigger": "onaction", "focusable": True}),
    Area("focus-1", Rect(1, 22, 6, 5), {"silent": True}),
    Area("focus-2", Rect(8, 22, 6, 5), {"silent": True}),
    Area("whiteboard", Rect(4, 28, 6, 4), {"openWebsite": "https://excalidraw.com/", "openWebsiteTrigger": "onaction"}),
    Area("pet-cat-1", Rect(17, 7, 3, 3)),
    Area("pet-cat-2", Rect(43, 29, 2, 3)),
    Area("pet-shiba", Rect(29, 33, 3, 3)),
    Area("koi-pond", Rect(47, 17, 7, 6)),
    Area("omikuji", Rect(47, 30, 3, 3)),
    *[
        Area(name, rect, {"openWebsite": PLAY_SITES[name], "openWebsiteTrigger": "onaction"})
        for name, rect in (
            ("play-go", Rect(50, 2, 3, 3)),
            ("play-arcade", Rect(52, 8, 2, 3)),
            ("play-lofi", Rect(44, 10, 3, 3)),
        )
        if name in PLAY_SITES
    ],
)

POPUPS = (
    Popup("catPopup1", Rect(17, 5, 3, 2)),
    Popup("catPopup2", Rect(43, 26, 2, 2)),
    Popup("shibaPopup", Rect(29, 31, 3, 2)),
    Popup("omikujiPopup", Rect(46, 26, 5, 2)),
)

LAYOUT = Layout(
    width=W,
    height=H,
    spawn=Rect(26, 35, 3, 2),
    fills=tuple(FLOORS + WALLS + KOI_FEEDING),
    stamps=tuple(FURNITURE),
    openings=OPENINGS,
    areas=AREAS,
    popups=POPUPS,
    hidden_layers=("koi-feeding",),
)
```

- [ ] **Step 6: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `60 passed`

ถ้า `test_spawn_free_and_every_area_reachable` fail ให้อ่านชื่อ area ใน error แล้วแก้พิกัดของ opening หรือเฟอร์นิเจอร์ที่ขวางทาง ห้ามแก้ test

- [ ] **Step 7: Commit**

```bash
git add tools/vocab.py tools/layout.py tools/tests/stub_registry.py tools/tests/test_layout.py
git commit -m "feat: add cozy office floor plan and vocabulary"
```

---

### Task 8: Script ของ map (`src/omikuji.ts`, `src/interactions.ts`, `src/main.ts`)

**Files:**
- Create: `src/omikuji.ts`, `src/interactions.ts`, `tests-ts/omikuji.test.ts`, `tools/tests/test_script_refs.py`
- Modify: `src/main.ts` (เขียนใหม่ทั้งไฟล์)

**Interfaces:**
- Consumes: ชื่อ area/popup/layer จาก `layout.LAYOUT` (Task 7); `check_script_refs` (Task 4)
- Produces:
  - `omikuji.ts`: `Fortune`, `FORTUNES`, `localDateKey(date)`, `hash32(text)`, `fortuneFor(playerName, date)`, `formatFortune(fortune)`
  - `interactions.ts`: `Interaction`, `INTERACTIONS`
  - `main.ts`: ผูก `INTERACTIONS` เข้ากับ `WA.room.area.onEnter/onLeave`

- [ ] **Step 1: เขียน test ของ omikuji ที่ fail**

`tests-ts/omikuji.test.ts` (อยู่นอก `src/` เพราะ `tsconfig.json` include `src` และ import ที่ลงท้าย `.ts` จะทำให้ `tsc` error):

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { FORTUNES, formatFortune, fortuneFor, hash32, localDateKey } from "../src/omikuji.ts";

test("has seven ranks from 大吉 to 大凶", () => {
    assert.deepEqual(
        FORTUNES.map((f) => f.rank),
        ["大吉", "中吉", "小吉", "吉", "末吉", "凶", "大凶"],
    );
});

test("hash32 is FNV-1a, deterministic and unsigned", () => {
    assert.equal(hash32("a"), 0xe40c292c);
    assert.equal(hash32("abc"), hash32("abc"));
    assert.ok(hash32("สวัสดี") >= 0);
});

test("localDateKey uses local calendar date with zero padding", () => {
    assert.equal(localDateKey(new Date(2026, 0, 5, 23, 59)), "2026-01-05");
});

test("same player keeps the same fortune for the whole local day", () => {
    const early = fortuneFor("Supakorn", new Date(2026, 9, 8, 0, 5));
    const late = fortuneFor("Supakorn", new Date(2026, 9, 8, 23, 55));
    assert.equal(early, late);
});

test("fortune changes across days", () => {
    const ranks = new Set<string>();
    for (let day = 1; day <= 30; day++) {
        ranks.add(fortuneFor("Supakorn", new Date(2026, 9, day, 12)).rank);
    }
    assert.ok(ranks.size > 1);
});

test("works with Thai and emoji player names", () => {
    const fortune = fortuneFor("สมชาย 🐱", new Date(2026, 9, 8, 12));
    assert.ok(FORTUNES.includes(fortune));
    assert.equal(fortune, fortuneFor("สมชาย 🐱", new Date(2026, 9, 8, 18)));
});

test("formatFortune shows rank, reading and Thai text", () => {
    assert.equal(formatFortune(FORTUNES[0]), "大吉 (ไดคิจิ)\nโชคดีที่สุด วันนี้ทำอะไรก็ราบรื่น");
});
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `node --test tests-ts/omikuji.test.ts`
Expected: FAIL ด้วย `ERR_MODULE_NOT_FOUND` สำหรับ `src/omikuji.ts`

- [ ] **Step 3: เขียน `src/omikuji.ts`**

ไฟล์นี้ห้ามใช้ `WA` และห้ามใช้ syntax ที่ Node type stripping ไม่รองรับ (เช่น `enum`, `namespace`):

```ts
export interface Fortune {
    readonly rank: string;
    readonly reading: string;
    readonly text: string;
}

export const FORTUNES: readonly Fortune[] = [
    { rank: "大吉", reading: "ไดคิจิ", text: "โชคดีที่สุด วันนี้ทำอะไรก็ราบรื่น" },
    { rank: "中吉", reading: "ชูคิจิ", text: "โชคดีมาก งานที่ค้างอยู่จะคืบหน้า" },
    { rank: "小吉", reading: "โชคิจิ", text: "โชคดีเล็กน้อย ความสุขอยู่ในเรื่องใกล้ตัว" },
    { rank: "吉", reading: "คิจิ", text: "โชคดี ทำตามแผนไปเรื่อยๆ" },
    { rank: "末吉", reading: "ซึเอะคิจิ", text: "โชคจะมาช่วงท้าย อดทนอีกนิด" },
    { rank: "凶", reading: "เคียว", text: "ระวังความผิดพลาดเล็กๆ ตรวจงานอีกรอบ" },
    { rank: "大凶", reading: "ไดเคียว", text: "วันนี้พักบ้าง ดื่มชาแล้วค่อยลุยใหม่" },
];

/** Calendar date in the player's local time zone, e.g. "2026-10-08". */
export function localDateKey(date: Date): string {
    const month = String(date.getMonth() + 1).padStart(2, "0");
    const day = String(date.getDate()).padStart(2, "0");
    return `${date.getFullYear()}-${month}-${day}`;
}

/** 32-bit FNV-1a over UTF-16 code units. */
export function hash32(text: string): number {
    let hash = 0x811c9dc5;
    for (let i = 0; i < text.length; i++) {
        hash ^= text.charCodeAt(i);
        hash = Math.imul(hash, 0x01000193);
    }
    return hash >>> 0;
}

/** Same player + same local day always draws the same fortune. */
export function fortuneFor(playerName: string, date: Date): Fortune {
    return FORTUNES[hash32(`${playerName}|${localDateKey(date)}`) % FORTUNES.length];
}

export function formatFortune(fortune: Fortune): string {
    return `${fortune.rank} (${fortune.reading})\n${fortune.text}`;
}
```

- [ ] **Step 4: รัน test ให้ผ่าน**

Run: `node --test tests-ts/omikuji.test.ts`
Expected: `ℹ pass 7` และ `ℹ fail 0`

- [ ] **Step 5: เขียน test ว่า script อ้างชื่อที่มีอยู่จริงใน map**

`tools/tests/test_script_refs.py`:

```python
from build_map import SCRIPT
from layout import LAYOUT
from mapgen.validate import SCRIPT_REF, check_script_refs


def test_interactions_reference_existing_map_objects():
    assert check_script_refs(LAYOUT, SCRIPT.read_text(encoding="utf-8")) == []


def test_every_script_area_is_referenced():
    refs = {name for kind, name in SCRIPT_REF.findall(SCRIPT.read_text(encoding="utf-8")) if kind == "area"}
    assert refs == {"pet-cat-1", "pet-cat-2", "pet-shiba", "koi-pond", "omikuji"}
```

Run: `.venv/bin/python -m pytest -q tools/tests/test_script_refs.py`
Expected: FAIL ด้วย `FileNotFoundError` สำหรับ `src/interactions.ts`

- [ ] **Step 6: เขียน `src/interactions.ts`**

ชื่อ area, popup และ layer ต้องอยู่ในรูป `key: "name"` (double quote) เพราะ `check_script_refs` อ่านด้วย regex นี้:

```ts
/// <reference types="@workadventure/iframe-api-typings" />
import { formatFortune, fortuneFor } from "./omikuji";

export type Interaction =
    | { kind: "popup"; area: string; popup: string; prompt: string; message: () => string }
    | { kind: "layer"; area: string; layer: string; prompt: string; durationMs: number };

export const INTERACTIONS: readonly Interaction[] = [
    {
        kind: "popup",
        area: "pet-cat-1",
        popup: "catPopup1",
        prompt: "กด SPACE เพื่อลูบแมว",
        message: () => "にゃ〜 แมวหลับตาพริ้ม ครางเบาๆ",
    },
    {
        kind: "popup",
        area: "pet-cat-2",
        popup: "catPopup2",
        prompt: "กด SPACE เพื่อลูบแมว",
        message: () => "にゃん! แมวกลิ้งตัวโชว์พุง",
    },
    {
        kind: "popup",
        area: "pet-shiba",
        popup: "shibaPopup",
        prompt: "กด SPACE เพื่อลูบชิบะ",
        message: () => "わん! ชิบะกระดิกหางต้อนรับ",
    },
    {
        kind: "layer",
        area: "koi-pond",
        layer: "koi-feeding",
        prompt: "กด SPACE เพื่อให้อาหารปลาคาร์ฟ",
        durationMs: 3000,
    },
    {
        kind: "popup",
        area: "omikuji",
        popup: "omikujiPopup",
        prompt: "กด SPACE เพื่อเสี่ยงเซียมซี おみくじ",
        message: () => formatFortune(fortuneFor(WA.player.name, new Date())),
    },
];
```

- [ ] **Step 7: เขียน `src/main.ts` ใหม่ทั้งไฟล์**

```ts
/// <reference types="@workadventure/iframe-api-typings" />
import { INTERACTIONS, type Interaction } from "./interactions";

type ActionMessage = ReturnType<typeof WA.ui.displayActionMessage>;
type Popup = ReturnType<typeof WA.ui.openPopup>;

WA.onInit()
    .then(() => {
        for (const interaction of INTERACTIONS) {
            bind(interaction);
        }
    })
    .catch((e) => console.error(e));

function bind(interaction: Interaction): void {
    let actionMessage: ActionMessage | undefined;
    let popup: Popup | undefined;

    const closePopup = (): void => {
        popup?.close();
        popup = undefined;
    };

    const run = (): void => {
        if (interaction.kind === "popup") {
            closePopup();
            popup = WA.ui.openPopup(interaction.popup, interaction.message(), [
                { label: "ปิด", className: "primary", callback: () => closePopup() },
            ]);
        } else {
            WA.room.showLayer(interaction.layer);
            setTimeout(() => WA.room.hideLayer(interaction.layer), interaction.durationMs);
        }
    };

    WA.room.area.onEnter(interaction.area).subscribe(() => {
        actionMessage = WA.ui.displayActionMessage({ message: interaction.prompt, callback: run });
    });

    WA.room.area.onLeave(interaction.area).subscribe(() => {
        actionMessage?.remove();
        actionMessage = undefined;
        closePopup();
    });
}
```

- [ ] **Step 8: ตรวจ type และรัน test ทั้งหมด**

Run: `npx tsc`
Expected: ไม่มี output (exit code 0)

Run: `.venv/bin/python -m pytest -q && node --test tests-ts/omikuji.test.ts`
Expected: `62 passed` และ `ℹ fail 0`

- [ ] **Step 9: Commit**

```bash
git add src/omikuji.ts src/interactions.ts src/main.ts tests-ts/omikuji.test.ts tools/tests/test_script_refs.py
git commit -m "feat: add pet, koi and omikuji interactions"
```

---

### Task 9: Sprite สำรอง (`draw_custom.py`)

**Files:**
- Create: `tools/draw_custom.py`, `tilesets/custom/cozy_custom.png` (generated)
- Test: `tools/tests/test_draw_custom.py`

**Interfaces:**
- Consumes: `TILE`, `Tileset` (Task 2)
- Produces:
  - `CUSTOM_TILESET: Tileset` (name `cozy_custom`, 8 columns x 3 rows)
  - `CUSTOM_TILES: dict[str, int]`, `CUSTOM_OBJECTS: dict[str, Grid]`, `CUSTOM_ANIMATIONS: dict[str, tuple[tuple[str, int], ...]]`
  - `paint(path: Path) -> None`

- [ ] **Step 1: เขียน test ที่ fail**

`tools/tests/test_draw_custom.py`:

```python
from PIL import Image

from draw_custom import CUSTOM_ANIMATIONS, CUSTOM_OBJECTS, CUSTOM_TILES, CUSTOM_TILESET, paint
from mapgen.model import TILE
from vocab import ANIMATED, OBJECT_SIZES


def test_sheet_size(tmp_path):
    out = tmp_path / "custom.png"
    paint(out)
    with Image.open(out) as image:
        assert image.size == (CUSTOM_TILESET.columns * TILE, CUSTOM_TILESET.rows * TILE)


def test_every_named_tile_has_pixels(tmp_path):
    out = tmp_path / "custom.png"
    paint(out)
    with Image.open(out) as image:
        sheet = image.convert("RGBA")
    for name, tile_id in CUSTOM_TILES.items():
        x, y = (tile_id % CUSTOM_TILESET.columns) * TILE, (tile_id // CUSTOM_TILESET.columns) * TILE
        alpha_max = sheet.crop((x, y, x + TILE, y + TILE)).getchannel("A").getextrema()[1]
        assert alpha_max > 0, f"tile '{name}' is empty"


def test_ids_fit_in_tileset():
    assert all(0 <= i < CUSTOM_TILESET.tilecount for i in CUSTOM_TILES.values())
    assert len(set(CUSTOM_TILES.values())) == len(CUSTOM_TILES)


def test_objects_match_vocab_sizes():
    for name, grid in CUSTOM_OBJECTS.items():
        assert (len(grid[0]), len(grid)) == OBJECT_SIZES[name]
        assert all(cell in CUSTOM_TILES for row in grid for cell in row)


def test_animations_cover_pets_and_koi():
    assert set(CUSTOM_ANIMATIONS) == set(ANIMATED)
    for frames in CUSTOM_ANIMATIONS.values():
        assert len(frames) == 2 and all(frame in CUSTOM_TILES for frame, _ in frames)
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_draw_custom.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'draw_custom'`

- [ ] **Step 3: เขียน `tools/draw_custom.py`**

```python
"""Draws fallback sprites (pets, koi, play corner) into tilesets/custom/cozy_custom.png.

Usage: .venv/bin/python tools/draw_custom.py
"""
from __future__ import annotations

from pathlib import Path
from typing import Callable

from PIL import Image, ImageDraw

from mapgen.model import TILE, Grid, Tileset

ROOT = Path(__file__).resolve().parent.parent
COLUMNS, ROWS = 8, 3
CUSTOM_TILESET = Tileset(name="cozy_custom", image="tilesets/custom/cozy_custom.png",
                         columns=COLUMNS, tilecount=COLUMNS * ROWS)

CUSTOM_TILES: dict[str, int] = {
    "cat": 0, "cat_f2": 1, "shiba": 2, "shiba_f2": 3, "koi": 4, "koi_f2": 5, "koi_feed": 6, "go_board": 7,
    "radio": 8, "arcade_top": 9, "pingpong_tl": 10, "pingpong_tm": 11, "pingpong_tr": 12,
    "arcade_bottom": 17, "pingpong_bl": 18, "pingpong_bm": 19, "pingpong_br": 20,
}
CUSTOM_OBJECTS: dict[str, Grid] = {
    "arcade": (("arcade_top",), ("arcade_bottom",)),
    "pingpong_table": (("pingpong_tl", "pingpong_tm", "pingpong_tr"), ("pingpong_bl", "pingpong_bm", "pingpong_br")),
}
CUSTOM_ANIMATIONS: dict[str, tuple[tuple[str, int], ...]] = {
    "cat": (("cat", 600), ("cat_f2", 600)),
    "shiba": (("shiba", 400), ("shiba_f2", 400)),
    "koi": (("koi", 500), ("koi_f2", 500)),
}

INK = (58, 42, 36, 255)
ORANGE = (232, 150, 72, 255)
TAN = (214, 150, 86, 255)
CREAM = (250, 236, 212, 255)
WHITE = (250, 250, 245, 255)
RED = (200, 60, 50, 255)
WATER = (92, 150, 170, 255)
RIPPLE = (150, 196, 210, 255)
FOOD = (190, 140, 70, 255)
WOOD = (196, 150, 98, 255)
DARK_WOOD = (120, 82, 52, 255)
GREEN = (46, 120, 84, 255)
NAVY = (44, 52, 92, 255)
SCREEN = (120, 220, 200, 255)

Painter = Callable[[ImageDraw.ImageDraw, int, int], None]


def _cat(tail_up: bool) -> Painter:
    def paint_cat(d: ImageDraw.ImageDraw, x: int, y: int) -> None:
        tail = (x + 25, y + 22, x + 29, y + 12) if tail_up else (x + 25, y + 26, x + 30, y + 28)
        d.line(tail, fill=INK, width=3)
        d.ellipse((x + 6, y + 14, x + 26, y + 28), fill=ORANGE, outline=INK)
        d.polygon([(x + 9, y + 9), (x + 10, y + 3), (x + 13, y + 7)], fill=ORANGE, outline=INK)
        d.polygon([(x + 15, y + 7), (x + 18, y + 3), (x + 19, y + 9)], fill=ORANGE, outline=INK)
        d.ellipse((x + 8, y + 6, x + 20, y + 18), fill=ORANGE, outline=INK)
        d.line((x + 11, y + 12, x + 13, y + 12), fill=INK)
        d.line((x + 15, y + 12, x + 17, y + 12), fill=INK)
    return paint_cat


def _shiba(tail_shift: int) -> Painter:
    def paint_shiba(d: ImageDraw.ImageDraw, x: int, y: int) -> None:
        d.arc((x + 20 + tail_shift, y + 10, x + 28 + tail_shift, y + 18), 90, 360, fill=TAN, width=3)
        d.ellipse((x + 5, y + 14, x + 25, y + 28), fill=TAN, outline=INK)
        d.ellipse((x + 9, y + 18, x + 17, y + 27), fill=CREAM)
        d.polygon([(x + 7, y + 9), (x + 8, y + 2), (x + 12, y + 6)], fill=TAN, outline=INK)
        d.polygon([(x + 14, y + 6), (x + 18, y + 2), (x + 19, y + 9)], fill=TAN, outline=INK)
        d.ellipse((x + 6, y + 5, x + 20, y + 17), fill=TAN, outline=INK)
        d.ellipse((x + 9, y + 11, x + 17, y + 17), fill=CREAM)
        d.point([(x + 10, y + 10), (x + 16, y + 10)], fill=INK)
        d.rectangle((x + 12, y + 13, x + 13, y + 14), fill=INK)
    return paint_shiba


def _koi(shift: int, food: bool = False) -> Painter:
    def paint_koi(d: ImageDraw.ImageDraw, x: int, y: int) -> None:
        d.rectangle((x, y, x + TILE - 1, y + TILE - 1), fill=WATER)
        d.arc((x + 2, y + 2, x + 14, y + 10), 200, 340, fill=RIPPLE)
        d.ellipse((x + 4 + shift, y + 8, x + 16 + shift, y + 14), fill=ORANGE)
        d.ellipse((x + 8 + shift, y + 9, x + 12 + shift, y + 12), fill=WHITE)
        d.ellipse((x + 16 - shift, y + 18, x + 28 - shift, y + 24), fill=WHITE)
        d.ellipse((x + 19 - shift, y + 19, x + 23 - shift, y + 22), fill=RED)
        if food:
            d.point([(x + 10, y + 4), (x + 14, y + 6), (x + 20, y + 5), (x + 24, y + 14), (x + 8, y + 26)], fill=FOOD)
            d.ellipse((x + 12, y + 14, x + 15, y + 17), fill=FOOD)
    return paint_koi


def _go_board(d: ImageDraw.ImageDraw, x: int, y: int) -> None:
    d.rectangle((x + 3, y + 3, x + 28, y + 28), fill=WOOD, outline=DARK_WOOD)
    for i in range(6, 27, 5):
        d.line((x + i, y + 6, x + i, y + 26), fill=DARK_WOOD)
        d.line((x + 6, y + i, x + 26, y + i), fill=DARK_WOOD)
    d.ellipse((x + 9, y + 9, x + 13, y + 13), fill=INK)
    d.ellipse((x + 19, y + 14, x + 23, y + 18), fill=WHITE, outline=INK)


def _radio(d: ImageDraw.ImageDraw, x: int, y: int) -> None:
    d.line((x + 22, y + 10, x + 27, y + 2), fill=INK, width=2)
    d.rectangle((x + 5, y + 10, x + 27, y + 27), fill=DARK_WOOD, outline=INK)
    d.ellipse((x + 8, y + 13, x + 18, y + 23), fill=CREAM, outline=INK)
    d.ellipse((x + 21, y + 15, x + 24, y + 18), fill=RED)


def _arcade(d: ImageDraw.ImageDraw, x: int, y: int) -> None:
    d.rectangle((x + 3, y + 2, x + 28, y + 62), fill=NAVY, outline=INK)
    d.rectangle((x + 7, y + 8, x + 24, y + 24), fill=INK)
    d.rectangle((x + 10, y + 12, x + 21, y + 20), fill=SCREEN)
    d.rectangle((x + 5, y + 32, x + 26, y + 38), fill=RED)
    d.line((x + 10, y + 30, x + 10, y + 35), fill=INK, width=2)
    d.ellipse((x + 17, y + 33, x + 20, y + 36), fill=WHITE)
    d.ellipse((x + 22, y + 33, x + 25, y + 36), fill=WHITE)


def _pingpong(d: ImageDraw.ImageDraw, x: int, y: int) -> None:
    d.rectangle((x + 6, y + 56, x + 10, y + 62), fill=INK)
    d.rectangle((x + 85, y + 56, x + 89, y + 62), fill=INK)
    d.rectangle((x + 2, y + 6, x + 93, y + 56), fill=GREEN, outline=WHITE, width=2)
    d.line((x + 4, y + 31, x + 91, y + 31), fill=WHITE)
    d.line((x + 47, y + 2, x + 47, y + 60), fill=CREAM, width=3)


PAINTS: list[tuple[int, Painter]] = [
    (CUSTOM_TILES["cat"], _cat(tail_up=False)),
    (CUSTOM_TILES["cat_f2"], _cat(tail_up=True)),
    (CUSTOM_TILES["shiba"], _shiba(0)),
    (CUSTOM_TILES["shiba_f2"], _shiba(-2)),
    (CUSTOM_TILES["koi"], _koi(0)),
    (CUSTOM_TILES["koi_f2"], _koi(3)),
    (CUSTOM_TILES["koi_feed"], _koi(1, food=True)),
    (CUSTOM_TILES["go_board"], _go_board),
    (CUSTOM_TILES["radio"], _radio),
    (CUSTOM_TILES["arcade_top"], _arcade),
    (CUSTOM_TILES["pingpong_tl"], _pingpong),
]


def paint(path: Path) -> None:
    sheet = Image.new("RGBA", (COLUMNS * TILE, ROWS * TILE), (0, 0, 0, 0))
    draw = ImageDraw.Draw(sheet)
    for tile_id, painter in PAINTS:
        painter(draw, (tile_id % COLUMNS) * TILE, (tile_id // COLUMNS) * TILE)
    path.parent.mkdir(parents=True, exist_ok=True)
    sheet.save(path)


if __name__ == "__main__":
    out = ROOT / CUSTOM_TILESET.image
    paint(out)
    print(f"draw_custom: wrote {out.relative_to(ROOT)}")
```

- [ ] **Step 4: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `67 passed`

- [ ] **Step 5: สร้างภาพจริงและดูด้วยตา**

Run: `.venv/bin/python tools/draw_custom.py`
Expected: `draw_custom: wrote tilesets/custom/cozy_custom.png`

เปิด `tilesets/custom/cozy_custom.png` ด้วย Read tool แล้วเช็คว่าเห็นแมว 2 ท่า, ชิบะ 2 ท่า, ปลาคาร์ฟ 3 ช่อง, กระดานโกะ, วิทยุ, ตู้ arcade (สูง 2 ช่อง) และโต๊ะปิงปอง (3x2 ช่อง) ถ้ารูปไหนดูไม่ออก ให้ปรับพิกัดใน painter ตัวนั้นแล้วรันซ้ำ

- [ ] **Step 6: Commit**

```bash
git add tools/draw_custom.py tools/tests/test_draw_custom.py tilesets/custom/cozy_custom.png
git commit -m "feat: draw fallback pet, koi and play-corner sprites"
```

---

### Task 10: ตรวจเว็บใน Play corner (`check_embed.py`)

**Files:**
- Create: `tools/check_embed.py`
- Test: `tools/tests/test_check_embed.py`
- Modify (ถ้ามีเว็บถูก block): `tools/layout.py` (`PLAY_SITES`)

**Interfaces:**
- Consumes: `layout.PLAY_SITES` (Task 7)
- Produces: `embed_blockers(headers: Mapping[str, str], origin: str) -> list[str]`, `fetch_headers(url) -> dict[str, str]`, `main() -> int`

- [ ] **Step 1: เขียน test ที่ fail**

`tools/tests/test_check_embed.py`:

```python
from check_embed import ORIGIN, embed_blockers


def test_no_headers_is_embeddable():
    assert embed_blockers({}, ORIGIN) == []


def test_x_frame_options_blocks():
    assert embed_blockers({"x-frame-options": "DENY"}, ORIGIN) == ["X-Frame-Options: deny"]
    assert embed_blockers({"x-frame-options": "sameorigin"}, ORIGIN) == ["X-Frame-Options: sameorigin"]


def test_frame_ancestors_none_blocks():
    csp = "default-src 'self'; frame-ancestors 'none'"
    assert embed_blockers({"content-security-policy": csp}, ORIGIN) == ["CSP frame-ancestors 'none'"]


def test_frame_ancestors_wildcard_or_origin_allows():
    assert embed_blockers({"content-security-policy": "frame-ancestors *"}, ORIGIN) == []
    assert embed_blockers({"content-security-policy": f"frame-ancestors 'self' {ORIGIN}"}, ORIGIN) == []
    assert embed_blockers({"content-security-policy": "frame-ancestors https:"}, ORIGIN) == []


def test_frame_ancestors_other_site_blocks():
    csp = "frame-ancestors https://example.com"
    assert embed_blockers({"content-security-policy": csp}, ORIGIN) == ["CSP frame-ancestors https://example.com"]


def test_origin():
    assert ORIGIN == "https://wa.alternatepro.com"
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_check_embed.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'check_embed'`

- [ ] **Step 3: เขียน `tools/check_embed.py`**

```python
"""Checks whether each play-corner site allows being shown in an iframe on our WorkAdventure domain.

Usage: .venv/bin/python tools/check_embed.py
Headers are necessary but not sufficient: a site may still refuse in JavaScript, so Task 13 checks in a browser too.
"""
from __future__ import annotations

import sys
import urllib.request
from typing import Mapping

ORIGIN = "https://wa.alternatepro.com"
USER_AGENT = "Mozilla/5.0 (Macintosh; Intel Mac OS X 14_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130 Safari/537.36"


def embed_blockers(headers: Mapping[str, str], origin: str) -> list[str]:
    reasons: list[str] = []
    frame_options = headers.get("x-frame-options", "").strip().lower()
    if frame_options in ("deny", "sameorigin"):
        reasons.append(f"X-Frame-Options: {frame_options}")
    for directive in headers.get("content-security-policy", "").split(";"):
        parts = directive.split()
        if not parts or parts[0].lower() != "frame-ancestors":
            continue
        sources = [p.lower() for p in parts[1:]]
        if "*" in sources or "https:" in sources or origin.lower() in sources:
            continue
        reasons.append(f"CSP frame-ancestors {' '.join(parts[1:])}")
    return reasons


def fetch_headers(url: str) -> dict[str, str]:
    request = urllib.request.Request(url, headers={"User-Agent": USER_AGENT})
    with urllib.request.urlopen(request, timeout=15) as response:
        headers = {key.lower(): value for key, value in response.headers.items()}
        policies = response.headers.get_all("content-security-policy") or []
    headers["content-security-policy"] = ";".join(policies)
    return headers


def main() -> int:
    from layout import PLAY_SITES

    blocked = 0
    for area, url in PLAY_SITES.items():
        try:
            reasons = embed_blockers(fetch_headers(url), ORIGIN)
        except OSError as error:
            reasons = [f"request failed: {error}"]
        status = "OK     " if not reasons else "BLOCKED"
        print(f"{status} {area:12} {url}  {'; '.join(reasons)}")
        blocked += bool(reasons)
    return 1 if blocked else 0


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `73 passed`

- [ ] **Step 5: ตรวจเว็บจริง**

Run: `.venv/bin/python tools/check_embed.py`
Expected: แต่ละบรรทัดขึ้น `OK` หรือ `BLOCKED` พร้อมเหตุผล

ถ้ามี `BLOCKED` ให้ลบ entry นั้นออกจาก `PLAY_SITES` ใน `tools/layout.py` (area และ test จะตามไปเอง) แล้วรันซ้ำจนทุกบรรทัดเป็น `OK` ห้ามลบของตกแต่ง (go_board, arcade, radio) ออกจาก `FURNITURE`

- [ ] **Step 6: รัน test และ commit**

Run: `.venv/bin/python -m pytest -q`
Expected: ผ่านทั้งหมด (จำนวนเท่าเดิม)

```bash
git add tools/check_embed.py tools/tests/test_check_embed.py tools/layout.py
git commit -m "feat: check play-corner sites allow iframe embedding"
```

ใน commit message หรือรายงานของ task ให้ระบุว่าเว็บไหนถูกตัดออก

---

### Task 11: เครื่องมือตรวจและแปลง tileset (`inspect_tileset.py`, `rescale_tileset.py`)

**Files:**
- Create: `tools/inspect_tileset.py`, `tools/rescale_tileset.py`
- Test: `tools/tests/test_tileset_tools.py`

**Interfaces:**
- Consumes: `TILE` (Task 2)
- Produces:
  - `describe(path, tile) -> tuple[int, int, int]` (columns, rows, tilecount; raise `ValueError` ถ้าขนาดไม่ลงตัว)
  - `contact_sheet(path, tile, out, scale=2) -> Path`
  - `rescale(src, dst, src_tile, dst_tile=TILE) -> tuple[int, int]`

- [ ] **Step 1: เขียน test ที่ fail**

`tools/tests/test_tileset_tools.py`:

```python
import pytest
from PIL import Image

from inspect_tileset import contact_sheet, describe
from rescale_tileset import rescale


def make_sheet(path, size, color=(200, 30, 30, 255)):
    image = Image.new("RGBA", size, (0, 0, 0, 0))
    image.paste(color, (0, 0, 8, 8))
    image.save(path)
    return path


def test_describe(tmp_path):
    assert describe(make_sheet(tmp_path / "a.png", (96, 64)), 32) == (3, 2, 6)


def test_describe_rejects_uneven_size(tmp_path):
    with pytest.raises(ValueError, match="not a multiple of 32"):
        describe(make_sheet(tmp_path / "a.png", (100, 64)), 32)


def test_contact_sheet_is_scaled(tmp_path):
    out = contact_sheet(make_sheet(tmp_path / "a.png", (96, 64)), 32, tmp_path / "out" / "a.sheet.png")
    with Image.open(out) as image:
        assert image.size == (192, 128)


def test_rescale_16_to_32_keeps_pixels_sharp(tmp_path):
    src = make_sheet(tmp_path / "s.png", (32, 16))
    size = rescale(src, tmp_path / "d" / "s.png", 16)
    assert size == (64, 32)
    with Image.open(tmp_path / "d" / "s.png") as image:
        assert image.convert("RGBA").getpixel((15, 15)) == (200, 30, 30, 255)


def test_rescale_48_to_32(tmp_path):
    src = make_sheet(tmp_path / "s.png", (96, 48))
    assert rescale(src, tmp_path / "d.png", 48) == (64, 32)


def test_rescale_rejects_uneven_size(tmp_path):
    with pytest.raises(ValueError):
        rescale(make_sheet(tmp_path / "s.png", (50, 48)), tmp_path / "d.png", 48)
```

- [ ] **Step 2: รัน test ให้เห็นว่า fail**

Run: `.venv/bin/python -m pytest -q tools/tests/test_tileset_tools.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'inspect_tileset'`

- [ ] **Step 3: เขียน `tools/inspect_tileset.py`**

```python
"""Prints tileset dimensions and writes a contact sheet with tile ids drawn on every tile.

Usage: .venv/bin/python tools/inspect_tileset.py --tile 32 tilesets/guttykreum32/*.png
Sheets are written to tools/out/<name>.sheet.png
"""
from __future__ import annotations

import argparse
from pathlib import Path

from PIL import Image, ImageDraw

ROOT = Path(__file__).resolve().parent.parent


def describe(path: Path, tile: int) -> tuple[int, int, int]:
    with Image.open(path) as image:
        width, height = image.size
    if width % tile or height % tile:
        raise ValueError(f"{path} is {width}x{height}, not a multiple of {tile}")
    columns, rows = width // tile, height // tile
    return columns, rows, columns * rows


def contact_sheet(path: Path, tile: int, out: Path, scale: int = 2) -> Path:
    columns, rows, _ = describe(path, tile)
    with Image.open(path) as source:
        art = source.convert("RGBA").resize((source.width * scale, source.height * scale), Image.Resampling.NEAREST)
    sheet = Image.new("RGBA", art.size, (40, 40, 48, 255))
    sheet.alpha_composite(art)
    draw = ImageDraw.Draw(sheet)
    step = tile * scale
    for tile_id in range(columns * rows):
        x, y = (tile_id % columns) * step, (tile_id // columns) * step
        draw.rectangle((x, y, x + step - 1, y + step - 1), outline=(255, 0, 255, 160))
        draw.text((x + 2, y + 1), str(tile_id), fill=(255, 255, 0, 255))
    out.parent.mkdir(parents=True, exist_ok=True)
    sheet.convert("RGB").save(out)
    return out


def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--tile", type=int, default=32)
    parser.add_argument("images", nargs="+", type=Path)
    args = parser.parse_args()
    for path in args.images:
        columns, rows, count = describe(path, args.tile)
        out = contact_sheet(path, args.tile, ROOT / "tools" / "out" / f"{path.stem}.sheet.png")
        print(f"{path}: columns={columns} rows={rows} tilecount={count} sheet={out.relative_to(ROOT)}")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: เขียน `tools/rescale_tileset.py`**

```python
"""Rescales every PNG under SRC_DIR from --from px tiles to 32 px tiles, keeping relative paths.

Usage: .venv/bin/python tools/rescale_tileset.py --from 48 tilesets/guttykreum tilesets/guttykreum32
Integer upscales use nearest-neighbor (stays sharp); anything else uses Lanczos (pixel art gets softer).
"""
from __future__ import annotations

import argparse
from pathlib import Path

from PIL import Image

from mapgen.model import TILE


def rescale(src: Path, dst: Path, src_tile: int, dst_tile: int = TILE) -> tuple[int, int]:
    with Image.open(src) as image:
        image = image.convert("RGBA")
        if image.width % src_tile or image.height % src_tile:
            raise ValueError(f"{src} is {image.width}x{image.height}, not a multiple of {src_tile}")
        size = (image.width // src_tile * dst_tile, image.height // src_tile * dst_tile)
        method = Image.Resampling.NEAREST if dst_tile % src_tile == 0 else Image.Resampling.LANCZOS
        result = image.resize(size, method)
    dst.parent.mkdir(parents=True, exist_ok=True)
    result.save(dst)
    return size


def main() -> None:
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("--from", dest="src_tile", type=int, required=True)
    parser.add_argument("src_dir", type=Path)
    parser.add_argument("dst_dir", type=Path)
    args = parser.parse_args()
    for src in sorted(args.src_dir.rglob("*.png")):
        dst = args.dst_dir / src.relative_to(args.src_dir)
        size = rescale(src, dst, args.src_tile)
        print(f"{src} -> {dst} ({size[0]}x{size[1]})")


if __name__ == "__main__":
    main()
```

- [ ] **Step 5: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: `79 passed`

- [ ] **Step 6: Commit**

```bash
git add tools/inspect_tileset.py tools/rescale_tileset.py tools/tests/test_tileset_tools.py
git commit -m "feat: add tileset inspection and rescale tools"
```

---

### Task 12: Map tile ของ pack (`tiles.py`) — ต้องมี pack ก่อน

**Files:**
- Create: `tools/tiles.py`
- Test: `tools/tests/test_tiles_data.py`

**Interfaces:**
- Consumes: `CUSTOM_*` (Task 9); `vocab` (Task 7); `check_images` (Task 4); `build`, `ROOT`, `SCRIPT` (Task 6); `describe`, `contact_sheet`, `rescale` (Task 11)
- Produces: `tiles.REGISTRY: Registry`

- [ ] **Step 1: Gate — รอ pack จากผู้ใช้**

เช็คว่ามีไฟล์:

```bash
find tilesets/guttykreum -name '*.png' | head -50
```

ถ้าไม่มีไฟล์ ให้หยุดและขอให้ผู้ใช้ซื้อและ unzip pack ไว้ที่ `maps/cozy-office/tilesets/guttykreum/` ห้ามทำ step ต่อไปจนกว่าจะมีไฟล์

- [ ] **Step 2: หาขนาด tile และแปลงเป็น 32x32**

อ่าน README และไฟล์ license ใน pack ก่อน ถ้า license ห้ามใช้งานแบบนี้ ให้หยุดและแจ้งผู้ใช้

ลองแต่ละขนาด:

```bash
for t in 16 32 48; do .venv/bin/python tools/inspect_tileset.py --tile $t tilesets/guttykreum/<sheet>.png; done
```

เปิดไฟล์ `tools/out/<sheet>.sheet.png` ด้วย Read tool ขนาดที่ถูกคือขนาดที่เส้นกรอบสีชมพูตรงกับขอบของของในภาพ

- ถ้า 32: ใช้ `tilesets/guttykreum/` ตรงๆ ในขั้นถัดไป
- ถ้า 16: `.venv/bin/python tools/rescale_tileset.py --from 16 tilesets/guttykreum tilesets/guttykreum32`
- ถ้า 48: `.venv/bin/python tools/rescale_tileset.py --from 48 tilesets/guttykreum tilesets/guttykreum32` แล้ว **หยุด** แสดง sheet ที่แปลงแล้วสัก 1 ไฟล์ให้ผู้ใช้ดูและอนุมัติความคมชัดก่อน (spec ข้อ 5)

- [ ] **Step 3: สร้าง contact sheet ของไฟล์ที่ใช้จริง**

```bash
.venv/bin/python tools/inspect_tileset.py --tile 32 tilesets/guttykreum32/*.png
```

(ถ้าเป็น 32 อยู่แล้ว ใช้ `tilesets/guttykreum/*.png`) จด `columns` และ `tilecount` ของแต่ละไฟล์

- [ ] **Step 4: เขียน test ที่ fail**

`tools/tests/test_tiles_data.py`:

```python
from build_map import ROOT, SCRIPT, build
from layout import LAYOUT
from mapgen.validate import check_images
from tiles import REGISTRY
from vocab import ANIMATED, OBJECT_SIZES, TILE_NAMES


def test_every_vocab_tile_is_mapped():
    assert [name for name in TILE_NAMES if name not in REGISTRY.tiles] == []


def test_every_vocab_object_has_declared_size():
    for name, size in OBJECT_SIZES.items():
        obj = REGISTRY.objects.get(name)
        assert obj is not None, f"object '{name}' is not mapped"
        assert (obj.width, obj.height) == size, f"object '{name}' is {obj.width}x{obj.height}, vocab says {size}"


def test_pets_and_koi_are_animated():
    assert [name for name in ANIMATED if name not in REGISTRY.animations] == []


def test_tileset_images_match_declared_sizes():
    assert check_images(REGISTRY, ROOT) == []


def test_full_map_builds():
    tmj, errors = build(LAYOUT, REGISTRY, SCRIPT.read_text(encoding="utf-8"), ROOT)
    assert errors == []
    assert tmj is not None
```

Run: `.venv/bin/python -m pytest -q tools/tests/test_tiles_data.py`
Expected: FAIL ด้วย `ModuleNotFoundError: No module named 'tiles'`

- [ ] **Step 5: เขียน `tools/tiles.py`**

โครงไฟล์คงที่ตามนี้ ส่วนที่ต้องเติมคือ `PACK_TILESETS`, `PACK_TILES`, `PACK_OBJECTS` (และ `PACK_ANIMATIONS` ถ้า pack มีสัตว์ที่ขยับได้) โดยอ่าน tile id จาก contact sheet ใน Step 3:

```python
"""Maps every vocab name to a tile. Pack tiles override the drawn fallbacks from draw_custom.py."""
from draw_custom import CUSTOM_ANIMATIONS, CUSTOM_OBJECTS, CUSTOM_TILES, CUSTOM_TILESET
from mapgen.model import ObjectDef, Registry, TileRef, Tileset

SPECIAL = Tileset(name="WA_Special_Zones", image="tilesets/WA_Special_Zones.png",
                  columns=6, tilecount=12, collides_ids=(2,))

# One Tileset per pack PNG. columns/tilecount come from tools/inspect_tileset.py.
PACK_TILESETS: tuple[Tileset, ...] = (
    # Tileset(name="gk_floors", image="tilesets/guttykreum32/<file>.png", columns=<n>, tilecount=<n>),
)

# Vocab name -> tile id read from tools/out/<file>.sheet.png
PACK_TILES: dict[str, TileRef] = {
    # "floor_tatami": TileRef("gk_floors", <id>),
}

# Multi-tile objects. Grid shape must equal vocab.OBJECT_SIZES[name]; use None for empty cells.
# Put the parts that should cover the player (tree canopy, top of tall furniture) in `above`.
PACK_OBJECTS: dict[str, ObjectDef] = {
    # "desk": ObjectDef(base=(("gk_desk_l", "gk_desk_r"),)),
}

# Only needed when PACK_TILES overrides an animated vocab name (cat, shiba, koi).
PACK_ANIMATIONS: dict[str, tuple[tuple[str, int], ...]] = {}

TILES: dict[str, TileRef] = {
    "start_marker": TileRef(SPECIAL.name, 1),
    "collision_marker": TileRef(SPECIAL.name, 2),
    **{name: TileRef(CUSTOM_TILESET.name, tile_id) for name, tile_id in CUSTOM_TILES.items()},
    **PACK_TILES,
}

OBJECTS: dict[str, ObjectDef] = {
    **{name: ObjectDef(base=grid) for name, grid in CUSTOM_OBJECTS.items()},
    **PACK_OBJECTS,
}

ANIMATIONS = {
    **{name: frames for name, frames in CUSTOM_ANIMATIONS.items() if name not in PACK_TILES},
    **PACK_ANIMATIONS,
}

REGISTRY = Registry(
    tilesets=(SPECIAL, CUSTOM_TILESET, *PACK_TILESETS),
    tiles=TILES,
    objects=OBJECTS,
    animations=ANIMATIONS,
)
```

กฎการเติม:
1. ทุกชื่อใน `vocab.TILE_NAMES` ที่ไม่อยู่ใน `CUSTOM_TILES` ต้องมีใน `PACK_TILES`
2. ทุกชื่อใน `vocab.OBJECT_SIZES` ที่ไม่อยู่ใน `CUSTOM_OBJECTS` ต้องมีใน `PACK_OBJECTS` ชิ้นส่วนย่อยของ object ให้ตั้งชื่อขึ้นต้นด้วย `gk_` แล้วใส่ใน `PACK_TILES` ด้วย
3. ถ้า pack ไม่มีของตรงตัว ให้ใช้ของที่ใกล้ที่สุด แล้วเขียน comment ข้างบรรทัดนั้นว่าใช้อะไรแทน
4. ถ้าของใน pack มีขนาดไม่ตรงกับ `OBJECT_SIZES` ให้แก้ขนาดใน `vocab.py` แล้วรัน `test_layout.py` ถ้า fail ให้ปรับพิกัดใน `layout.py`
5. ถ้าไม่มีของที่ใช้แทนได้เลย ให้หยุดและถามผู้ใช้ว่าจะซื้อ pack เพิ่ม หรือให้วาดเพิ่มใน `draw_custom.py`
6. ถ้า pack มีแมว ชิบะ หรือปลาที่ดีกว่า sprite สำรอง ใส่ใน `PACK_TILES` ได้ แต่ต้องใส่ `PACK_ANIMATIONS` ของชื่อนั้นด้วย

- [ ] **Step 6: รัน test ให้ผ่าน**

Run: `.venv/bin/python -m pytest -q`
Expected: ผ่านทั้งหมด (`84 passed`)

- [ ] **Step 7: Commit**

```bash
git add tools/tiles.py tools/tests/test_tiles_data.py tools/vocab.py tools/layout.py
git status --short tilesets/guttykreum tilesets/guttykreum32
git commit -m "feat: map GuttyKreum pack tiles to vocab"
```

Expected: `git status` ของสองโฟลเดอร์ pack ไม่มี output (ถูก ignore) ถ้ามีไฟล์ pack โผล่มา ห้าม commit ให้แก้ `.gitignore` ก่อน

---

### Task 13: Build, preview และตรวจใน browser

**Files:**
- Create: `cozy-office.tmj` (generated)

**Interfaces:**
- Consumes: ทุกอย่างก่อนหน้า
- Produces: `cozy-office.tmj` ที่ผ่านการตรวจ, ภาพหน้าจอให้ผู้ใช้ review

- [ ] **Step 1: Build map**

Run: `.venv/bin/python tools/build_map.py`
Expected: `build_map: wrote cozy-office.tmj (56x38 tiles)`

- [ ] **Step 2: Build ด้วย wa-map-optimizer**

Run: `npm run buildmap`
Expected: exit code 0; มี `dist/cozy-office.tmj` และ tileset ที่ใช้ใน `dist/`

- [ ] **Step 3: เปิด preview**

Run (background): `npm run dev`
เปิด `http://localhost:5173/` แล้วกดลิงก์ทดสอบของ `cozy-office.tmj`

- [ ] **Step 4: ตรวจตาม checklist และ capture หน้าจอ**

ใช้ browser tool (claude-in-chrome) เดินดูและ capture หน้าจอทุกโซน:

1. เกิดที่ genkan และ layer `start`/`collisions` มองไม่เห็น
2. เดินชนผนัง เฟอร์นิเจอร์ และบ่อน้ำแล้วไปต่อไม่ได้ ถ้าเดินทะลุได้ แปลว่า WorkAdventure ไม่อ่าน collision จาก layer ที่ซ่อนไว้ ให้แก้ `_tile_layer("collisions", ..., visible=False)` ใน `mapgen/tmj.py` เป็น `visible=True` แล้วเพิ่ม `"opacity": 0` และแก้ test `test_visibility` ให้ตรงกัน จากนั้น build ใหม่
3. เข้าห้องประชุมทั้ง 2 ห้อง: ขึ้น prompt ให้กด SPACE แล้ว Jitsi เปิด
4. เข้า focus booth: เสียงจากคนนอกห้องถูกตัด
5. Whiteboard และ Play corner ทุกจุด: กด SPACE แล้วเว็บโหลดได้ ถ้าหน้าว่างหรือขึ้น error ให้ตัด entry นั้นออกจาก `PLAY_SITES` (เหมือน Task 10 Step 5)
6. แมว 2 ตัวและชิบะ: ขยับตัว (animation), กด SPACE แล้ว popup ขึ้น, กดซ้ำแล้ว popup ไม่ซ้อนกัน, เดินออกแล้ว popup ปิด
7. บ่อปลา: ปลาว่าย, กด SPACE แล้วเห็นอาหารปลา 3 วินาทีแล้วหายไป
8. Omikuji: กด SPACE ได้ผล, กดซ้ำวันเดียวกันได้ผลเดิม
9. กิ่งซากุระและส่วนบนของเฟอร์นิเจอร์สูงบังตัวละครเมื่อเดินผ่านด้านหลัง
10. ไม่มี error ใน browser console

- [ ] **Step 5: Gate — ให้ผู้ใช้ review**

ส่งภาพหน้าจอให้ผู้ใช้ดู ถามว่ามีโซนไหนอยากปรับไหม ถ้าผู้ใช้ขอแก้ ให้แก้ `layout.py` (หรือ `tiles.py` ถ้าเป็นเรื่องภาพ) รัน `.venv/bin/python -m pytest -q` แล้วกลับไป Step 1 ห้ามแก้ `cozy-office.tmj` ด้วยมือ เพราะจะถูกเขียนทับตอน build ครั้งต่อไป

- [ ] **Step 6: Commit**

```bash
git add cozy-office.tmj tools/
git commit -m "feat: build cozy office map"
```

---

### Task 14: Upload ขึ้น `wa.alternatepro.com`

**Files:**
- Modify (บน server): `.env` ของ WorkAdventure (`MAP_STORAGE_API_TOKEN`, `START_ROOM_URL`)
- Create (ในเครื่อง, gitignored): `maps/cozy-office/.env.secret`

**Interfaces:**
- Consumes: `cozy-office.tmj` จาก Task 13
- Produces: map ที่ `https://wa.alternatepro.com/~/cozy-office/cozy-office.wam` และเป็นหน้าแรกของ server

ทุก step ใน task นี้แตะ server จริง ให้ผู้ใช้เป็นคนรันคำสั่งบน server หรืออนุมัติก่อนทุกครั้ง

- [ ] **Step 1: ตั้ง API token บน server (ผู้ใช้รัน)**

บน Ubuntu server ในโฟลเดอร์ WorkAdventure:

```bash
openssl rand -hex 32
```

ใส่ค่าที่ได้ใน `.env` ที่บรรทัด `MAP_STORAGE_API_TOKEN=<ค่าที่ได้>` แล้วรัน:

```bash
docker compose up -d
```

- [ ] **Step 2: Backup map-storage ก่อน upload (ผู้ใช้รัน)**

```bash
docker volume ls --format '{{.Name}}' | grep map-storage-data
docker run --rm -v <ชื่อ volume จากบรรทัดบน>:/data -v "$PWD":/backup alpine \
  tar czf /backup/map-storage-backup-$(date +%F).tgz -C /data .
ls -lh map-storage-backup-*.tgz
```

Expected: มีไฟล์ backup ขนาดมากกว่า 0

- [ ] **Step 3: Upload จากเครื่อง**

Run: `npm run upload`

ตอบคำถามของสคริปต์:
- Map storage URL: `https://wa.alternatepro.com/map-storage/`
- API key: ค่า `MAP_STORAGE_API_TOKEN` จาก Step 1
- Upload directory: `cozy-office`

Expected: upload สำเร็จ และสคริปต์แสดง URL ของ map ถ้า URL ที่ได้ไม่ใช่ `/~/cozy-office/cozy-office.wam` ให้ใช้ URL ที่สคริปต์แสดงใน step ถัดไปแทน

- [ ] **Step 4: ทดสอบ map บน server**

เปิด `https://wa.alternatepro.com/~/cozy-office/cozy-office.wam` แล้วทำ checklist ข้อ 1-10 ของ Task 13 Step 4 ซ้ำอีกครั้ง

- [ ] **Step 5: ตั้งเป็นหน้าแรก (ผู้ใช้รัน)**

บน server แก้ `.env`:

```
START_ROOM_URL=/~/cozy-office/cozy-office.wam
```

แล้วรัน:

```bash
docker compose up -d
curl -sI https://wa.alternatepro.com/ | head -1
```

Expected: `HTTP/2 200` และเปิด `https://wa.alternatepro.com/` แล้วเข้า cozy office

- [ ] **Step 6: Commit การแก้ไขที่เหลือ (ถ้ามี)**

```bash
git status --short
git add -A
git commit -m "chore: publish cozy office to wa.alternatepro.com"
```

ถ้า `git status` ไม่มีอะไรเปลี่ยน ให้ข้าม commit
