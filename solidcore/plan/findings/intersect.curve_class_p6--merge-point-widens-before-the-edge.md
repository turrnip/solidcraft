# intersect.curve_class_p6: the merge's point step widens the new vertex before the edge (2026-11-28)

- **Unit:** `intersect.curve_class_p6`, through `topology::brep::p9::make_edge_in_face_with_vu`.
- **Cases (2):** `imprint_face_split_f0`, `imprint_face_split_f2`.
- **Values:** after the merge is refused for the width its edge would need (0.11804298874989479,
  past the 0.02 that Test #82 allows for a body of tolerance 1e-5), the new vertex still carries
  0.1180249887499, and the body has 9 vertices and 13 edges. The refused step's own value is
  absent.
- **Observation:** `merge_interval_mut` merges the curve's end point into the box edge before it
  makes the edge. That merge splits the box edge and glues two vertices, and the new vertex is
  already past the 0.02 width when `p9` is entered. A refusal inside `make_edge_in_face_with_vu`
  therefore prevents that function's edge and its own growth of the vertex, but it cannot unpick
  the widening the merge did before the call. The refusal is still right: the caller gets a typed
  error instead of a body the validator refuses with Test #82.
- **Suspected cause:** the point merge has no width test before it glues. The glue sets the
  survivor's tolerance to the gluing distance plus the deleted vertex's tolerance
  (`topology::brep::p5::glue_vertices`, `crates/solidcore/src/topology/brep/p5.rs:1794-1816`);
  the same §5.1 limit the validator reads (`vertices_to_clear`, Test #82) is checked after the
  merge, not before it.
- **Status:** open; follow-up to the width refusal in `MakeEdgeInFaceWithVU` (#21368). The
  refusal's own scope holds: no edge, no growth in `p9`.
