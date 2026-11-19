# Deck_Automation
Explanation of working :

Cells 1 to 9 do almost no actual work. They install libraries, set paths and define functions, which is like writing recipes without cooking anything. The real run starts in cell 10, with the single call to run_deck_update(...). Everything else happens from inside that call.
Phase 0: Preparation (cells 1–9, nothing is built yet)
Cells 1–2 install the libraries and restart Python.
Cell 3 does one real action: it deletes old decks from the output folder. It also sets three variables: VOLUME_CONFIG_PATH, VOLUME_TEMPLATE_PATH and VOLUME_OUTPUT_PATH (with today's date).
Cell 4 turns ANSI mode off for Spark SQL.
Cells 5–9 only import libraries and define functions, plus COMBINED_TABLE and the highlight color. Cell 6 is fully commented out, so it does nothing.
Phase 1: The run starts in cell 10
run_deck_update(config_path, template_path, output_path) runs these steps in order:
read_config(config_path) loads the whole Excel file into a pandas DataFrame config_df, one row per element.
fetch_all_data(config_df, 15) gets the data:
It picks out the rows that have SQL and removes duplicate queries.
It sends each distinct query to a thread pool of 15 workers. Each worker calls fetch_data, which runs spark.sql(q).toPandas() and unpacks the JSON when the result is a json_format_df column.
It returns data_cache, a dictionary mapping each config row number to its result DataFrame, kept in memory.
Presentation(template_path) opens the template as prs.
get_chart_excel_mapping(template_path) opens the PPTX as a zip and records which embedded Excel file belongs to which chart.
Loop over every config row. For each row:
It reads slide_num, static_text and show_last_label, and takes the row's DataFrame from data_cache.
It calls debug_wrap_update(...) (cell 8), which prints debug info and then calls update_element(...):
find_shape(prs, slide_num, shape_name) finds the shape on that slide.
Based on element_type, it calls one updater:
element_typeFunction calledWhat it does
chart
update_chart_data (plus add_last_point_data_labels if show_last_label is on)
Writes the new values into the chart XML
textbox
update_textbox
Replaces the text and colors ▲/▼
exec summary
update_exec_summary_textbox
Rebuilds bold headings and bullets
table
update_table
Fills cells row by row
If the update worked, it adds one to the success count. For charts, it also saves the DataFrame in chart_data_map (for highlights) and in chart_data_for_excel (for the embedded Excel).
If it failed, it adds one to the failure count and moves on. One broken element doesn't stop the run.
Speaker notes. It runs a spark.sql query on COMBINED_TABLE for slide_insights and writes a "Narrative:" section with bullets into each slide's notes. If this step fails, it prints an error and continues.
Save.
prs.save("/tmp/_deck_temp.pptx")
update_embedded_excel_in_pptx(temp, final, chart_data_for_excel) rewrites each chart's Excel sheet inside the zip and produces /tmp/_deck_final.pptx.
The bytes are copied to VOLUME_OUTPUT_PATH on the Volume and the temp files are deleted.
It prints the success and failure counts and returns chart_data_map.
Phase 2: After the deck is built (cells 11–14)
Cell 11 is commented out and skipped.
Cell 12 overwrites the narrative_deck_latest table with the file name, link and timestamp, so Genie can tell users where the deck is.
Cell 13 checks that the file exists and prints its size.
Cell 14 calls dbutils.notebook.exit(...) with the path and "success". The notebook stops here, so cells 15–17 never run in a full run.
The whole path in one line
Cell 10 calls run_deck_update, which runs: read_config → fetch_all_data (parallel fetch_data) → open template → get_chart_excel_mapping → for each row: debug_wrap_update → update_element → find_shape → chart/textbox/exec summary/table updater → speaker notes → save to /tmp → update_embedded_excel_in_pptx → copy to Volume. Then cells 12 to 14 record the deck location, check the file and exit.
What's in the code but never runs
All the highlight functions (run_deck_update_with_highlights, read_highlights_from_db, apply_all_highlights, the hl_* helpers) are defined but never called, because cell 10 calls run_deck_update. They would only run if cell 10 called run_deck_update_with_highlights instead. That version calls run_deck_update first, then reopens the saved deck, draws the highlights and saves it again.
