# -*- coding: utf-8 -*-
"""
FreeCAD macro based on the user's latest working luer-lock/cuboid version.

Revision v23:
  - Keeps the reinforced luer-lock + cuboid + flush edge logic.
  - Closes the distal end of the extension tube.
  - Replaces the rounded slots with four more rectangular side openings.
  - The rectangular openings begin exactly at the distal closed-end/cap
    interface of the extension tube and run 5 mm back along the tube body.
  - The cutters still enter at 45 degrees to the tube axis and are distributed
    at 90 degree intervals around the tube.

Notes:
  - The original STEP geometry is preserved for the luer-lock shape.
  - The extension tube is fused first, then the central lumen is cut, stopping
    short of the distal tip so the tube end remains closed.
  - The side openings are cut afterwards so the lumen can vent through them.
"""

import os
import math

import FreeCAD as App
import Part

try:
    import FreeCADGui as Gui
except Exception:
    Gui = None


BUILD_META = {}


class P:
    # STEP file source.
    source_step_path = ""
    source_step_filename = "luer-lock-cap.step"

    # Geometry kept from source STEP.
    keep_side = "z_min"
    keep_height_from_side = 10.0
    crop_z_min_from_original_bottom = None
    crop_z_max_from_original_bottom = None

    # Central open passage.
    through_hole_diameter = 4.0

    # Extension tube.
    add_hole_extension_tube = True
    extension_tube_length = 50.0
    extension_tube_outer_diameter = 4.5
    extension_tube_overlap = 3.60
    extension_side = "z_max"

    # New in v23: close distal end and add 4 angled rectangular-ish openings at
    # the very distal/bottom end of the extension tube, where the tube meets the cap.
    close_extension_tube_end = True
    extension_end_cap_thickness = 1.0
    add_extension_side_openings = True
    extension_side_opening_count = 4
    extension_side_opening_style = "rectangular"
    extension_side_opening_diameter = 1.4   # rectangular opening width
    extension_side_opening_angle_deg = 90.0
    extension_side_opening_height = 20.0     # rectangular opening axial height, starting at cap interface
    extension_side_opening_cut_length = 22.0  # radial/angled cutter depth
    extension_side_opening_slot_samples = 9  # ignored for rectangular style; kept for compatibility
    extension_side_opening_overlap_into_cap = .0
    extension_side_opening_outside_margin = 1.4

    # Surrounding cuboid.
    add_surrounding_cuboid = True
    cuboid_length = 13.0
    cuboid_width = 13.0
    cuboid_height = 18.0
    cuboid_luer_top_side = "z_max"
    cuboid_top_above_luer_lock = 8.0
    cuboid_height_change_side = "same_as_luer_top"
    cuboid_height_reference = 22.0

    # Cylindrical hollow in cuboid.
    cuboid_hollow_cylinder_diameter = 12.0
    cuboid_hollow_height_above_luer_lock = 6.0
    cuboid_hollow_height_below_luer_lock = 6.0
    cuboid_hollow_open_side = "opposite_to_luer_top"
    cuboid_cut_hollow_through_full_height = False

    # Reinforcement.
    add_structural_reinforcement_sleeve = True
    structural_sleeve_outer_diameter = 11.2
    structural_sleeve_height = 8.0
    structural_sleeve_overlap_into_hollow = 1.2
    add_structural_backing_collar = True
    structural_backing_collar_diameter = 13.2
    structural_backing_collar_height = 2.0

    # Flush cuboid edge/rim on hollow side.
    add_cuboid_edge_on_hollow_side = True
    cuboid_edge_height = 1.0
    cuboid_edge_thickness = 1.0
    cuboid_edge_inward_overlap = 0.25

    # Optional small lead-in/relief.
    add_top_lead_in = False
    top_lead_in_diameter = 3.2
    top_lead_in_depth = 1.0
    add_bottom_lead_in = False
    bottom_lead_in_diameter = 3.0
    bottom_lead_in_depth = 0.8

    normalize_final_placement = True
    show_source_reference = False
    source_reference_transparency = 80
    export_step = False
    export_stl = False
    export_dir = os.path.join(os.path.expanduser("~"), "luer_lock_exports")


def console(msg):
    App.Console.PrintMessage(str(msg) + "\n")


def safe_remove_splitter(shape):
    try:
        return shape.removeSplitter()
    except Exception:
        return shape


def fuse_many(shapes):
    shapes = [s for s in shapes if s is not None and not s.isNull()]
    if not shapes:
        raise ValueError("No valid shapes to fuse.")
    out = shapes[0]
    for s in shapes[1:]:
        out = out.fuse(s)
    return safe_remove_splitter(out)


def cut_many(shape, cutters):
    out = shape
    for c in cutters:
        if c is not None and not c.isNull():
            out = out.cut(c)
    return safe_remove_splitter(out)


def find_source_step_path():
    candidates = []
    if getattr(P, "source_step_path", ""):
        candidates.append(P.source_step_path)
    fname = getattr(P, "source_step_filename", "luer-lock-cap.step")
    try:
        macro_dir = os.path.dirname(os.path.abspath(__file__))
        candidates.append(os.path.join(macro_dir, fname))
    except Exception:
        pass
    try:
        candidates.append(os.path.join(os.getcwd(), fname))
    except Exception:
        pass
    try:
        candidates.append(os.path.join(App.getUserMacroDir(True), fname))
    except Exception:
        pass
    candidates.append(os.path.join("/mnt/data", fname))

    checked = []
    for path in candidates:
        if not path:
            continue
        path = os.path.abspath(os.path.expanduser(path))
        if path in checked:
            continue
        checked.append(path)
        if os.path.isfile(path):
            return path

    raise FileNotFoundError(
        "Could not find the STEP file. Set P.source_step_path to the full path, "
        "or put luer-lock-cap.step in the same folder as this macro. Checked:\n  "
        + "\n  ".join(checked)
    )


def import_step_as_shape(doc, step_path):
    import Import
    before = {obj.Name for obj in doc.Objects}
    Import.insert(step_path, doc.Name)
    imported = [obj for obj in doc.Objects if obj.Name not in before]
    shapes = []
    for obj in imported:
        if hasattr(obj, "Shape") and obj.Shape is not None and not obj.Shape.isNull():
            shapes.append(obj.Shape.copy())
    if not shapes:
        raise RuntimeError("STEP import produced no Part shapes.")
    source_shape = fuse_many(shapes)
    if not getattr(P, "show_source_reference", False):
        for obj in imported:
            try:
                doc.removeObject(obj.Name)
            except Exception:
                pass
    else:
        for obj in imported:
            try:
                obj.Label = "Source_STEP_reference"
                if Gui is not None and hasattr(obj, "ViewObject"):
                    obj.ViewObject.Transparency = int(P.source_reference_transparency)
            except Exception:
                pass
    return source_shape


def make_crop_box_for_desired_side(shape):
    bb = shape.BoundBox
    z_min_original = bb.ZMin
    z_max_original = bb.ZMax
    exact_min = getattr(P, "crop_z_min_from_original_bottom", None)
    exact_max = getattr(P, "crop_z_max_from_original_bottom", None)
    if exact_min is not None or exact_max is not None:
        z0 = z_min_original + (0.0 if exact_min is None else float(exact_min))
        z1 = z_min_original + (bb.ZLength if exact_max is None else float(exact_max))
        if z1 <= z0:
            raise ValueError("crop_z_max_from_original_bottom must be larger than crop_z_min_from_original_bottom.")
    else:
        keep_h = float(getattr(P, "keep_height_from_side", 10.0))
        side = str(getattr(P, "keep_side", "z_min")).strip().lower()
        if side in {"z_min", "bottom", "lower", "min"}:
            z0 = z_min_original
            z1 = min(z_max_original, z_min_original + keep_h)
        elif side in {"z_max", "top", "upper", "max"}:
            z0 = max(z_min_original, z_max_original - keep_h)
            z1 = z_max_original
        else:
            raise ValueError("P.keep_side must be 'z_min'/'bottom' or 'z_max'/'top'.")
    margin = 20.0
    length = max(1.0, bb.XLength + 2.0 * margin)
    width = max(1.0, bb.YLength + 2.0 * margin)
    height = (z1 - z0) + 2.0
    box = Part.makeBox(length, width, height, App.Vector(bb.XMin - margin, bb.YMin - margin, z0 - 1.0))
    return box, z0, z1


def crop_to_desired_luerlock_side(source_shape):
    crop_box, z0, z1 = make_crop_box_for_desired_side(source_shape)
    cropped = source_shape.common(crop_box)
    cropped = safe_remove_splitter(cropped)
    if cropped.isNull() or cropped.BoundBox.ZLength <= 0:
        raise RuntimeError("Cropping failed. Increase keep_height_from_side or switch keep_side.")
    console(f"Keeping original STEP Z range: {z0:.3f} to {z1:.3f} mm")
    return cropped


def central_axis_xy(shape):
    bb = shape.BoundBox
    return (bb.XMin + bb.XMax) / 2.0, (bb.YMin + bb.YMax) / 2.0


def cylinder_z(diameter, height, zmin, x, y):
    return Part.makeCylinder(float(diameter) / 2.0, float(height), App.Vector(float(x), float(y), float(zmin)), App.Vector(0, 0, 1))


def make_centered_box(length, width, height, zmin, x, y):
    return Part.makeBox(float(length), float(width), float(height), App.Vector(float(x - length / 2.0), float(y - width / 2.0), float(zmin)))


def _side_is_z_max(value):
    return str(value).strip().lower() in {"z_max", "top", "upper", "max", "up"}


def _side_is_z_min(value):
    return str(value).strip().lower() in {"z_min", "bottom", "lower", "min", "down"}


def add_extension_tube(shape):
    if not getattr(P, "add_hole_extension_tube", True):
        BUILD_META["has_extension"] = False
        return shape

    length = float(getattr(P, "extension_tube_length", 40.0))
    outer_d = float(getattr(P, "extension_tube_outer_diameter", 4.4))
    overlap = max(0.0, float(getattr(P, "extension_tube_overlap", 0.60)))
    hole_d = float(getattr(P, "through_hole_diameter", 2.2))
    if outer_d <= hole_d + 0.1:
        console("Warning: extension_tube_outer_diameter is only slightly larger than through_hole_diameter.")

    bb = shape.BoundBox
    x, y = central_axis_xy(shape)
    side = str(getattr(P, "extension_side", "z_min")).strip().lower()
    if side in {"z_min", "bottom", "lower", "min", "down"}:
        zmin = bb.ZMin - length
        height = length + overlap
        distal_end_z = zmin
        join_end_z = bb.ZMin + overlap
    elif side in {"z_max", "top", "upper", "max", "up"}:
        zmin = bb.ZMax - overlap
        height = length + overlap
        distal_end_z = zmin + height
        join_end_z = bb.ZMax - overlap
    else:
        raise ValueError("P.extension_side must be 'z_min'/'bottom' or 'z_max'/'top'.")

    tube = cylinder_z(outer_d, height, zmin, x, y)

    BUILD_META.update({
        "has_extension": True,
        "extension_side": side,
        "extension_outer_diameter": outer_d,
        "extension_zmin": zmin,
        "extension_zmax": zmin + height,
        "extension_distal_end_z": distal_end_z,
        "extension_join_end_z": join_end_z,
        "extension_center_x": x,
        "extension_center_y": y,
        "cropped_zmin": bb.ZMin,
        "cropped_zmax": bb.ZMax,
    })
    return fuse_many([shape, tube])


def _cuboid_hollow_opens_toward_z_max(luer_top_side):
    mode = str(getattr(P, "cuboid_hollow_open_side", "opposite_to_luer_top")).strip().lower()
    luer_ref_is_zmax = _side_is_z_max(luer_top_side)
    if mode in {"z_max", "top", "upper", "max", "up"}:
        return True
    if mode in {"z_min", "bottom", "lower", "min", "down"}:
        return False
    if mode in {"same", "same_as_luer_top", "same_as_reference", "luer_top"}:
        return luer_ref_is_zmax
    if mode in {"opposite", "opposite_to_luer_top", "other", "other_end", "opposite_as_reference"}:
        return not luer_ref_is_zmax
    raise ValueError("cuboid_hollow_open_side must be valid.")


def _cuboid_height_changes_toward_z_max(luer_top_side):
    mode = str(getattr(P, "cuboid_height_change_side", "opposite_to_luer_top")).strip().lower()
    luer_ref_is_zmax = _side_is_z_max(luer_top_side)
    if mode in {"z_max", "top", "upper", "max", "up"}:
        return True
    if mode in {"z_min", "bottom", "lower", "min", "down"}:
        return False
    if mode in {"same", "same_as_luer_top", "same_as_reference", "luer_top"}:
        return luer_ref_is_zmax
    if mode in {"opposite", "opposite_to_luer_top", "other", "other_end", "opposite_as_reference"}:
        return not luer_ref_is_zmax
    raise ValueError("cuboid_height_change_side must be valid.")


def _cuboid_z_range_from_height_policy(luer_top_z, luer_top_side, height, top_offset):
    H = float(height)
    ref_H = float(getattr(P, "cuboid_height_reference", 22.0))
    luer_ref_is_zmax = _side_is_z_max(luer_top_side)
    change_toward_zmax = _cuboid_height_changes_toward_z_max(luer_top_side)
    if luer_ref_is_zmax:
        luer_side_face_z = float(luer_top_z) + float(top_offset)
        if change_toward_zmax:
            zmin = luer_side_face_z - ref_H
            zmax = zmin + H
        else:
            zmax = luer_side_face_z
            zmin = zmax - H
    else:
        luer_side_face_z = float(luer_top_z) - float(top_offset)
        if change_toward_zmax:
            zmin = luer_side_face_z
            zmax = zmin + H
        else:
            zmax = luer_side_face_z + ref_H
            zmin = zmax - H
    return zmin, zmax


def _add_flush_edge_to_cuboid_face(cuboid, x, y, cuboid_zmin, cuboid_zmax, open_toward_zmax):
    if not getattr(P, "add_cuboid_edge_on_hollow_side", True):
        return cuboid
    L = float(getattr(P, "cuboid_length", 14.0))
    W = float(getattr(P, "cuboid_width", 14.0))
    edge_h = float(getattr(P, "cuboid_edge_height", 1.0))
    edge_t = float(getattr(P, "cuboid_edge_thickness", 1.0))
    overlap = float(getattr(P, "cuboid_edge_inward_overlap", 0.25))
    if edge_h <= 0 or edge_t <= 0:
        return cuboid

    # Frame is flush with the cuboid face: it extends outward in XY but occupies
    # the same face depth range, not above the face.
    if open_toward_zmax:
        frame_zmin = cuboid_zmax - edge_h
    else:
        frame_zmin = cuboid_zmin
    outer = make_centered_box(L + 2.0 * edge_t, W + 2.0 * edge_t, edge_h + overlap, frame_zmin - (overlap if open_toward_zmax else 0.0), x, y)
    inner = make_centered_box(L, W, edge_h + overlap + 0.2, frame_zmin - 0.1 - (overlap if open_toward_zmax else 0.0), x, y)
    frame = safe_remove_splitter(outer.cut(inner))
    return fuse_many([cuboid, frame])


def add_surrounding_cuboid(shape, luer_reference_shape):
    if not getattr(P, "add_surrounding_cuboid", True):
        return shape

    L = float(getattr(P, "cuboid_length", 14.0))
    W = float(getattr(P, "cuboid_width", 14.0))
    H = float(getattr(P, "cuboid_height", 22.0))
    x, y = central_axis_xy(luer_reference_shape)
    ref_bb = luer_reference_shape.BoundBox
    side = str(getattr(P, "cuboid_luer_top_side", "z_max")).strip().lower()
    top_offset = float(getattr(P, "cuboid_top_above_luer_lock", 8.0))
    luer_top_z = ref_bb.ZMax if _side_is_z_max(side) else ref_bb.ZMin
    cuboid_zmin, cuboid_zmax = _cuboid_z_range_from_height_policy(luer_top_z, side, H, top_offset)
    cuboid = make_centered_box(L, W, H, cuboid_zmin, x, y)

    well_d = float(getattr(P, "cuboid_hollow_cylinder_diameter", 12.0))
    well_h = float(getattr(P, "cuboid_hollow_height_above_luer_lock", 8.0)) + float(getattr(P, "cuboid_hollow_height_below_luer_lock", 6.0))
    open_toward_zmax = _cuboid_hollow_opens_toward_z_max(side)
    if open_toward_zmax:
        well_z0 = cuboid_zmax - well_h
    else:
        well_z0 = cuboid_zmin
    if bool(getattr(P, "cuboid_cut_hollow_through_full_height", False)):
        well_z0 = cuboid_zmin - 1.0
        well_h = H + 2.0
    well = cylinder_z(well_d, well_h + 0.2, well_z0 - 0.1, x, y)
    cuboid = cut_many(cuboid, [well])

    # Reinforcement sleeve / backing collar.
    if bool(getattr(P, "add_structural_reinforcement_sleeve", True)):
        sleeve_d = min(float(getattr(P, "structural_sleeve_outer_diameter", 11.2)), min(L, W) - 0.4)
        sleeve_h = float(getattr(P, "structural_sleeve_height", 8.0))
        sleeve_overlap = max(0.0, float(getattr(P, "structural_sleeve_overlap_into_hollow", 1.2)))
        reinforcers = []
        if open_toward_zmax:
            sleeve_z0 = max(cuboid_zmin, well_z0 - sleeve_h)
            sleeve_h_eff = (well_z0 - sleeve_z0) + sleeve_overlap
        else:
            sleeve_z0 = well_z0 + well_h - sleeve_overlap
            sleeve_h_eff = min(sleeve_h + sleeve_overlap, cuboid_zmax - sleeve_z0)
        if sleeve_h_eff > 0.2:
            reinforcers.append(cylinder_z(sleeve_d, sleeve_h_eff, sleeve_z0, x, y))
        if bool(getattr(P, "add_structural_backing_collar", True)):
            collar_d = min(float(getattr(P, "structural_backing_collar_diameter", 13.2)), min(L, W) - 0.2)
            collar_h = float(getattr(P, "structural_backing_collar_height", 2.0))
            if collar_h > 0.1:
                if open_toward_zmax:
                    collar_z0 = max(cuboid_zmin, well_z0 - collar_h)
                else:
                    collar_z0 = min(cuboid_zmax - collar_h, well_z0 + well_h)
                reinforcers.append(cylinder_z(collar_d, collar_h, collar_z0, x, y))
        if reinforcers:
            cuboid = fuse_many([cuboid] + reinforcers)

    cuboid = _add_flush_edge_to_cuboid_face(cuboid, x, y, cuboid_zmin, cuboid_zmax, open_toward_zmax)

    BUILD_META.update({
        "cuboid_zmin": cuboid_zmin,
        "cuboid_zmax": cuboid_zmax,
        "cuboid_hollow_open_toward_zmax": open_toward_zmax,
    })

    return fuse_many([shape, cuboid])


def _make_parallelepiped_from_corner(p0, a_vec, b_vec, c_vec):
    """Return a solid parallelepiped from one corner and three edge vectors."""
    p1 = p0 + a_vec
    p2 = p0 + a_vec + b_vec
    p3 = p0 + b_vec
    p4 = p0 + c_vec
    p5 = p0 + a_vec + c_vec
    p6 = p0 + a_vec + b_vec + c_vec
    p7 = p0 + b_vec + c_vec

    faces = []
    for pts in (
        [p0, p1, p2, p3, p0],
        [p4, p7, p6, p5, p4],
        [p0, p4, p5, p1, p0],
        [p1, p5, p6, p2, p1],
        [p2, p6, p7, p3, p2],
        [p3, p7, p4, p0, p3],
    ):
        faces.append(Part.Face(Part.makePolygon(pts)))
    return Part.Solid(Part.makeShell(faces))


def _make_side_opening_cutters(shape):
    """Create four angled rectangular side openings at the closed distal end.

    The opening begins at the cap/lumen interface, not 5 mm away from it. The
    5 mm parameter is the axial opening height. A small overlap into the cap is
    used so the rectangular window reaches the very bottom internally, while the
    distal end face itself remains closed because the central lumen stops before
    the cap.
    """
    if not (getattr(P, "add_hole_extension_tube", True) and getattr(P, "add_extension_side_openings", True) and BUILD_META.get("has_extension", False)):
        return []

    count = int(getattr(P, "extension_side_opening_count", 4))
    if count <= 0:
        return []

    opening_width = float(getattr(P, "extension_side_opening_width", getattr(P, "extension_side_opening_diameter", 1.4)))
    opening_height = float(getattr(P, "extension_side_opening_height", 5.0))
    angle_deg = float(getattr(P, "extension_side_opening_angle_deg", 45.0))
    cut_length = float(getattr(P, "extension_side_opening_cut_length", 12.0))
    cap_overlap = max(0.0, float(getattr(P, "extension_side_opening_overlap_into_cap", 0.20)))
    outside_margin = max(0.1, float(getattr(P, "extension_side_opening_outside_margin", 1.4)))

    if opening_width <= 0 or opening_height <= 0 or cut_length <= 0:
        return []

    outer_r = 0.5 * float(BUILD_META["extension_outer_diameter"])
    x = float(BUILD_META["extension_center_x"])
    y = float(BUILD_META["extension_center_y"])
    side = BUILD_META["extension_side"]
    distal_z = float(BUILD_META["extension_distal_end_z"])
    cap_t = float(getattr(P, "extension_end_cap_thickness", 1.0)) if getattr(P, "close_extension_tube_end", True) else 0.0

    # Body direction is from the closed cap back toward the luer body.
    if side in {"z_max", "top", "upper", "max", "up"}:
        cap_interface_z = distal_z - cap_t
        body_sign = -1.0
    else:
        cap_interface_z = distal_z + cap_t
        body_sign = 1.0

    # The cutter direction is 45 degrees to the tube axis in the radial-Z plane.
    # It starts outside the tube and travels inward while also leaning back
    # toward the tube body. The rectangular opening's axial edge starts at the
    # cap interface and extends opening_height back along the tube.
    angle = math.radians(angle_deg)
    radial_component = math.sin(angle)
    axial_component = body_sign * math.cos(angle)

    cutters = []
    start_offset = outer_r + outside_margin
    h_len = opening_height + cap_overlap

    for i in range(count):
        theta = 2.0 * math.pi * i / float(count)
        radial = App.Vector(math.cos(theta), math.sin(theta), 0)
        tangent = App.Vector(-math.sin(theta), math.cos(theta), 0)

        # Cutter travels from outside inward, with the specified 45-degree tilt.
        cut_axis = App.Vector(
            -math.cos(theta) * radial_component,
            -math.sin(theta) * radial_component,
            axial_component,
        )

        # Axial edge starts just inside the cap and goes back into the tube body.
        # For z_max extension this is downward; for z_min extension upward.
        h_axis = App.Vector(0, 0, body_sign)

        start_center = App.Vector(
            x + radial.x * start_offset,
            y + radial.y * start_offset,
            cap_interface_z - body_sign * cap_overlap,
        )

        # Build the cutter as a rectangular-ish parallelepiped.  This gives a
        # flatter-sided window than the previous rounded/cylindrical slot.
        a_vec = App.Vector(cut_axis.x * cut_length, cut_axis.y * cut_length, cut_axis.z * cut_length)
        b_vec = App.Vector(tangent.x * opening_width, tangent.y * opening_width, tangent.z * opening_width)
        c_vec = App.Vector(h_axis.x * h_len, h_axis.y * h_len, h_axis.z * h_len)
        p0 = start_center - App.Vector(b_vec.x * 0.5, b_vec.y * 0.5, b_vec.z * 0.5)
        cutters.append(_make_parallelepiped_from_corner(p0, a_vec, b_vec, c_vec))

    return cutters

def cut_through_hole(shape):
    bb = shape.BoundBox
    x, y = central_axis_xy(shape)
    cutters = []

    # Main central lumen.
    if getattr(P, "add_hole_extension_tube", True) and BUILD_META.get("has_extension", False) and getattr(P, "close_extension_tube_end", True):
        cap_t = float(getattr(P, "extension_end_cap_thickness", 1.0))
        side = BUILD_META["extension_side"]
        zmin_tube = float(BUILD_META["extension_zmin"])
        zmax_tube = float(BUILD_META["extension_zmax"])
        if side in {"z_max", "top", "upper", "max", "up"}:
            hole_z0 = bb.ZMin - 2.0
            hole_z1 = zmax_tube - cap_t
        else:
            hole_z0 = zmin_tube + cap_t
            hole_z1 = bb.ZMax + 2.0
        cutters.append(cylinder_z(P.through_hole_diameter, max(0.1, hole_z1 - hole_z0), hole_z0, x, y))
    else:
        cutters.append(cylinder_z(P.through_hole_diameter, bb.ZLength + 4.0, bb.ZMin - 2.0, x, y))

    if getattr(P, "add_top_lead_in", False):
        cutters.append(cylinder_z(P.top_lead_in_diameter, float(P.top_lead_in_depth) + 0.1, bb.ZMax - float(P.top_lead_in_depth), x, y))
    if getattr(P, "add_bottom_lead_in", False):
        cutters.append(cylinder_z(P.bottom_lead_in_diameter, float(P.bottom_lead_in_depth) + 0.1, bb.ZMin - 0.05, x, y))

    # Four angled side openings near distal end of extension tube.
    cutters.extend(_make_side_opening_cutters(shape))

    return cut_many(shape, cutters)


def normalize_shape(shape):
    if not getattr(P, "normalize_final_placement", True):
        return shape
    bb = shape.BoundBox
    cx = (bb.XMin + bb.XMax) / 2.0
    cy = (bb.YMin + bb.YMax) / 2.0
    z0 = bb.ZMin
    moved = shape.copy()
    moved.translate(App.Vector(-cx, -cy, -z0))
    return moved


def add_feature(doc, name, shape, color=None, transparency=None):
    obj = doc.addObject("Part::Feature", name)
    obj.Label = name
    obj.Shape = shape
    if Gui is not None and hasattr(obj, "ViewObject"):
        if color is not None:
            obj.ViewObject.ShapeColor = color
        if transparency is not None:
            obj.ViewObject.Transparency = transparency
    return obj


def maybe_export(doc, obj):
    if not getattr(P, "export_step", False) and not getattr(P, "export_stl", False):
        return
    os.makedirs(P.export_dir, exist_ok=True)
    if getattr(P, "export_step", False):
        import Import
        path = os.path.join(P.export_dir, "luer_lock_v23.step")
        Import.export([obj], path)
        console(f"Exported STEP: {path}")
    if getattr(P, "export_stl", False):
        import Mesh
        path = os.path.join(P.export_dir, "luer_lock_v23.stl")
        Mesh.export([obj], path)
        console(f"Exported STL: {path}")


def create_document():
    step_path = find_source_step_path()
    console(f"Using source STEP: {step_path}")
    doc = App.newDocument("Luer_Lock_V23")
    source = import_step_as_shape(doc, step_path)
    cropped = crop_to_desired_luerlock_side(source)
    extended = add_extension_tube(cropped)
    cuboid_added = add_surrounding_cuboid(extended, cropped)
    holed = cut_through_hole(cuboid_added)
    final_shape = normalize_shape(holed)
    obj = add_feature(doc, "Luer_Lock_V23", final_shape, color=(0.72, 0.76, 0.78), transparency=0)
    doc.recompute()
    bb = final_shape.BoundBox
    console("Created luer-lock geometry with cuboid, closed-end extension tube, and 4 rectangular bottom side openings.")
    console(f"Final size: {bb.XLength:.3f} x {bb.YLength:.3f} x {bb.ZLength:.3f} mm")
    console(f"Through-hole diameter: {float(P.through_hole_diameter):.3f} mm")
    console(f"Extension tube OD={float(P.extension_tube_outer_diameter):.3f} mm, length={float(P.extension_tube_length):.3f} mm")
    console(f"Extension end closed: {bool(getattr(P, 'close_extension_tube_end', True))}, cap thickness={float(getattr(P, 'extension_end_cap_thickness', 1.0)):.3f} mm")
    console(f"Bottom rectangular openings: count={int(getattr(P, 'extension_side_opening_count', 4))}, width={float(getattr(P, 'extension_side_opening_diameter', 1.4)):.3f} mm, angle={float(getattr(P, 'extension_side_opening_angle_deg', 45.0)):.1f} deg, height from cap interface={float(getattr(P, 'extension_side_opening_height', 5.0)):.3f} mm")
    if Gui is not None:
        Gui.ActiveDocument.ActiveView.viewAxonometric()
        Gui.SendMsgToActiveView("ViewFit")
    maybe_export(doc, obj)
    return doc


if __name__ == "__main__":
    create_document()
