Updates: WR-469

#### WR-469 (update, 2026-11-28). The step has a width limit; a merge that would pass it is refused

`make_edge_in_face_with_vu` (`crates/solidcore/src/topology/brep/p9.rs`) grows each end vertex's
tolerance to twice its gap to the new edge's end, as the fork's fix does. It grew it with no width
limit: any edge end could be merged into a vertex, and the vertex made as wide as the distance to
that end, even past the width `validate_pointers` refuses whatever the vertex's ball clears. A
merge like that does not close a rounding or intersection gap; it absorbs an edge end that is
misplaced by more than the body's tolerance can cover (tangency-labs/solidcraft#21368).

The step now has the validator's own bound. Before the first edit (`b.add_edge_record`), for each
end `(vertex, t)` it computes the tolerance the merge would need, twice the farthest edge end at the
vertex with the new end (`merge_vertex_tolerance`), and refuses when that is past
`widest_vertex_tolerance(b) = GROWTH_PAST_RATIO * LARGE_SMALL_SIZE_RATIO * b.tolerance()`, the width
`vertices_to_clear` refuses in Test #82 (`crates/solidcore/src/topology/brep/validation/pointers.rs`).
The refusal is `TopologyFailure::VertexToleranceTooWide { needed, limit }`, at the vertex it would
have widened. No edge record is made and no vertex is widened. A merge that would not widen a
vertex does not refuse: the step only grows a tolerance, so a vertex already that wide is not this
merge's doing.

- **The measured case:** `intersect.curve_class_p6` `imprint_face_split_f0` and `_f2`. A unit box
  (body tolerance 1e-5) has a line from the origin along (1,2,0)/sqrt(5), length 1, whose far end is
  classified onto the edge (0,1,0)-(1,1,0) at t = 0.5, the point (0.5,1,0), 0.118 from the line's
  end. The merge puts its new vertex at (0.4736,0.9472,0), 0.059021 from each end it joins, and the
  step grows the vertex to twice that, 0.11804298874989479 (MEASURED). The widest this box accepts
  is 2 * 1000 * 1e-5 = 0.02, so the merge is now refused with needed 0.11804298874989479 and limit
  0.02 (MEASURED). The clearance was never the fault: 0.4736 is four vertex-ball radii from the box
  and `vertex_ball_reaches` is false, so only the width refused the body, in Test #82 ("Vertex
  tolerance is too wide to excuse by clearance").
- **Goldens:** the two rows move, from 181 values to the refusal's single status value: `values.out`
  becomes `[1001.0]` (MEASURED, the `SM_ERR` the typed failure carries), and `validate_pointers`
  refuses neither body because neither merge happens. `imprint_face_split_at_a_valid_gap_keeps_clear_of_the_box`
  keeps its row values: the same fixture with the line shortened to a 2.45e-5 gap needs 4.9e-5,
  still below 0.02, and builds as before.
- **Tests:** `imprint_face_split_past_the_width_limit_is_refused` replays the two rows (each is the
  single 1001) and calls `merge_interval_mut` directly: it asserts the typed refusal's `needed` and
  `limit`, the vertex it names, and that the box is untouched (8 vertices, 12 edges, every vertex
  tolerance still 1e-5). It fails without the guard, where the merge returns a body
  `validate_pointers` refuses with Test #82.
- **Why not re-classify:** the classification of the line's end onto the edge is right; the point
  (0.5,1,0) is on the edge and the shortfall is the line's own. What is wrong is the merge's growth
  rule, so the merge refuses instead of the classification deciding differently
  (tangency-labs/solidcraft#21368).
