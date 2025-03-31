# # Create/upload a new BLOB
POST `/api/v2/apps/{owner}/{app}/workspaces/{wsid}/blobs`

Creates a new BLOB with the uploaded binary data and metadata.

## Headers
| Key | Value |
| --- | --- |
| Authorization | Bearer {PrincipalToken} |
| Content-type | multipart/form-data |

### Parameters
| Parameter | Type | Description |
| --- | --- | --- |
| owner | string | name of a user who owns the application |
| app | string | name of an application |
| wsid | int64 | the ID of workspace |

## Body
BLOB and metadata.

Example request:
 ```
 POST /api/v2/apps/untill/airsbp3/workspaces/12344566789/blobs HTTP/1.1
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="image.jpg"
Content-Type: image/jpeg

<binary data here>
--boundary
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{
    "name": "Example Image",
    "tags": ["image", "example", "upload"],
    "description": "This is a sample image uploaded as a BLOB."
}
--boundary--
 ```

## Result
| Code | Description | Body |
| --- | --- | --- |
| 201 | Created | blobId and metadata, see example below |
| 413 | Payload Too Large | [error object](conventions.md#errors) |
| 415 | Unsupported Media Type | [error object](conventions.md#errors) |
| 400 | Bad Request | [error object](conventions.md#errors) |


Example response 201:
```
{
    "blobId": "1010231232123123",
    "name": "Example Image",
    "tags": ["image", "example", "upload"],
    "description": "This is a sample image uploaded as a BLOB.",
    "contentType": "image/jpeg",
    "size": 524288,  // Size in bytes
    "url": "https://federation.example.com/api/v2/users/untill/apps/airsbp3/workspaces/12344566789/blobs/1010231232123123"
}
```

