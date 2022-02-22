# ST_Point

## description

### Syntax

```Haskell
POINT ST_Point(DOUBLE x, DOUBLE y)
```

通过给定的 X 坐标值、Y 坐标值返回对应的 Point。
当前这个值只是在球面集合上有意义，X/Y 对应的是经度/纬度(longitude/latitude)。
> 直接 select ST_Point()会卡住，慎重！！！

## example

```Plain Text
MySQL > SELECT ST_AsText(ST_Point(24.7, 56.7));
+---------------------------------+
| st_astext(st_point(24.7, 56.7)) |
+---------------------------------+
| POINT (24.7 56.7)               |
+---------------------------------+
```

## keyword

ST_POINT, ST, POINT
