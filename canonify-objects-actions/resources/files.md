# Files — upload, attach, seed and show

A file's bytes live in the platform's blob store; its row in `_files` names
it (`file_…`), its name, type, size, uploader and upload time, and the record
that holds it. A record holds a file in one of three places:

| Where | Declare | Write it with | Shown by |
|---|---|---|---|
| A `file` property (a signed PDF, a floor plan, an item photo) | `"photo": { "kind": "mapped", "column": "photo", "type": "file", "accept"?: […], "max_bytes"?: n }` | the property's value: a `file_…` id in a create/update, or the file inline in `objects.import` | a `Fact`, `Meta`, `Column` or `Item` slot naming the property: the file by name (an image as a thumbnail), never its id |
| An attachment kind (a child type's rows) | `attachments.kinds.<kind>` on the parent ([object-property-types](object-property-types.md)) | `<child>.create` with the parent's id, the kind and a file id; or inline in `objects.import` of the child rows | `<Files of={record} />`: one block per kind |
| Attached to the record itself | `attachments: { accept?, max_bytes? }` on the type (optional rules) | `POST /v1/objects/<type>/<id>/files`; or `files` entries in `objects.import` | `<Files of={record} />`: an "Attachments" block |

Every read of a file is the reader's read of the record that holds it, and
of the `file` property when a property holds it: a reader whose policy
withholds the property (`columns.mode: "deny"`) is shown it as withheld —
no name, no id — and gets `403` from the download.

## Seed many files: `objects.import`

A seed carries its files in the rows, inline, and one call takes up to 500
rows and 10 MB of files (one file alone up to the 25 MB upload cap). The
import stores every file before it writes a row; a call of 150 small photos
is one call.

```bash
# rows.jsonl — a file property, and the record's own attachments
{"id": "itm_001", "name": "Banquet chair", "photo": {"$file": "photos/itm_001.png", "created_at": "2026-03-04T09:30:00.000Z"}}
{"id": "con_014", "client_id": "cli_3", "$attachments": [{"$file": "docs/con_014-signed.pdf", "created_at": "2026-03-04T09:30:00.000Z"}]}

canon objects import item --file rows.jsonl              # $file paths are read from the rows file's folder
canon objects import item --file rows.jsonl --files ./seedfiles
```

The CLI reads each `$file` and sends it inline; over REST or MCP send the same
call yourself:

```json
{
  "object_type": "item",
  "rows": [
    {
      "id": "itm_001",
      "name": "Banquet chair",
      "photo": { "name": "itm_001.png", "mime_type": "image/png", "base64": "iVBORw0K…", "created_at": "2026-03-04T09:30:00.000Z" }
    }
  ],
  "files": [
    { "row": 0, "name": "itm_001-invoice.pdf", "mime_type": "application/pdf", "base64": "JVBERi0x…" }
  ]
}
```

`POST /v1/actions/objects.import` with `{ "input": … }` (MCP: `actions.invoke`
with `name: "objects.import"`). A file a rule refuses refuses the call naming
its place — `rows[3].photo`, `files[2]` — and nothing is stored or imported.
A file the store fails to keep refuses the call `INTERNAL_ERROR` with
`details.code: "import_file_store_failed"`, `details.where` (`rows[3].photo`),
`details.name` and `details.retryable: true`: nothing was imported, so
re-run the call (with your own ids and `on_conflict: "skip"` a re-run is
safe). Files already uploaded (below) are placed by id instead: `"photo":
"file_01…"`, or `files: [{ "id": "file_01…", "row": 0 }]`.

## Upload one file: REST

Uploads are `multipart/form-data` with one part named `file`; the answer is
`{ "id": "file_01…", "content_type": "…", "size": n }`. The CLI has no upload
command — use these routes, or `objects.import` above for new rows.

```bash
# 1. Store it, held to a slot's rules before a byte is stored (optional
#    fields: object_type + property, object_type + kind, or action + input).
curl -X POST "$API/v1/files" -H "Authorization: Bearer $CANONIFY_API_KEY" \
  -F file=@signed.pdf -F object_type=contract -F property=signed_pdf
# 2. Write the id to the property: the write binds the file to the row.
canon contract.update --id con_014 --signed_pdf file_01JX…

# Attach a file to a record itself (its update grant decides):
curl -X POST "$API/v1/objects/contract/con_014/files" \
  -H "Authorization: Bearer $CANONIFY_API_KEY" -F file=@delivery-note.pdf
```

An uploaded file no row holds is its uploader's alone. Write it to a `file`
property (any write path binds it) or name it in a kind's child create; an
attached file is listed at once. Remove an attachment with `files.detach {
file_id }`; clear or replace a property with an update.

## Read and download

| Read | REST | CLI | MCP |
|---|---|---|---|
| A record's files | `GET /v1/objects/<type>/<id>/attachments` (`?kinds=`, `?include=`) | `canon objects attachments <type> <id> [--kinds k] [--include link]` | `objects.attachments` |
| Rows with their files named | `GET /v1/objects/<type>/<id>?link_titles=true`, `GET /v1/objects/<type>?link_titles=true` | `canon objects list <type> --link-titles` | `objects.get` / `objects.list` with `link_titles: true` |
| The bytes | `GET /v1/files/<id>` (your credentials), or `GET /v1/files/<id>/link` → `?ticket=` for a browser | — | — |

A titled read (link titles on) carries each `file` property's facts beside its
id as `<property>__file`: `{ id, name, mime_type, size_bytes, created_at }` —
only for a file the row holds, and only when you may read the property.
PNG, JPEG, GIF, WebP and PDF download inline unless `?download=1`; any other
type always as a download.

## Show them in SpringML

```jsx
<RecordPage type="contract">
  <Meta><Fact field="signed_pdf" /></Meta>            {/* the PDF by name, a link that opens it */}
  <Section title="Files"><Files of={record} /></Section>  {/* kinds, then the record's own Attachments */}
</RecordPage>
```

A `Column field="photo"` shows each row's image as a thumbnail with its name;
`<Item sub={[p.photo]} />` the same in a List. `<Files kinds={["photo"]} />`
shows those kinds only (no Attachments block).
