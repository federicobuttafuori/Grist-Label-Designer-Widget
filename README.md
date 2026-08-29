# Grist Label Designer Widget

A custom widget for Grist that allows you to design and print product labels directly from your table data.

## Features

### Visual Editor
- **Canvas editor** with drag-and-drop and zoom
- **Available elements**:
  - Text
  - Data fields (linked to Grist columns)
  - Images
  - QR codes
  - Barcodes
  - Lines and boxes
- **Label profiles** with customizable sizes
- **Multi-design** support for each profile

### Advanced Features
- **Data binding**: Link elements to Grist columns for automatic updates
- **Import/Export**: Save and load label layouts
- **Print preview**: Optimized for thermal printers
- **Print controls**: Gamma, contrast, and threshold tuning for black/white output
- **Text rules**: Regex-based transformations for automatic formatting

### Printing
- **Print**: one label, when the design has no sequence/wave fields
- **Print Sequence**: waves plus Print Single (count and weight), when the design uses sequences
- **Thermal optimization**: Controls tailored for thermal printers

## Using the widget in Grist

You do **not** need to publish or install anything yourself. The widget is already hosted here:

**[https://federicobuttafuori.github.io/Grist-Label-Designer-Widget/](https://federicobuttafuori.github.io/Grist-Label-Designer-Widget/)**

1. In your Grist document, add a widget to the page and choose **Custom widget** (or **Create custom widget**, depending on your Grist UI).
2. When asked for the widget URL, **paste the link above** into the **Custom widget URL** field.
3. Point the widget at the table you want and start designing.

## Usage

1. **Configure the Widget**:
   - Select the Grist table you want to use
   - Choose or create a label profile

2. **Design the Label**:
   - Add elements to the canvas
   - Link data fields to Grist columns
   - Configure properties in the sidebar

3. **Print**:
   - Use **Print** for a simple one-off label
   - Use **Print Sequence** when the design uses sequence or pack-wave fields
