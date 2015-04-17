# How to create the SBD.AP24\_ApacheHTTP unit. #

For reasons of copyright, I cannot post the SBD.AP24\_ApacheHTTP unit to the code repository. Instead, here are instructions for creating the unit.

  1. Copy D2010's unit ApacheTwoHTTP and name it SBD.AP24.ApacheHTTP. (In XE6 ApacheTwoHTTP was renamed Web.ApacheHTTP)
  1. Replace the unit reference to HTTPD to SBD.AP24.httpd
  1. Add a unit reference to Generics.Collections
  1. Add declaration
```pascal

const
ServerName_Idx = 600;
ServerVers_Idx = 601;
Handler_Idx    = 602;
```
  1. Rename TApacheRequest to TApacheTwoRequest and TApacheResponse to TApacheTwoResponse
  1. Add these private data members to class TApacheTwoRequest
```
    FRawHeaders: TStrings;
    FDummy: TStrings;
```
  1. Add these two public methods to the class
```
  public
    destructor Destroy; override;
    function GetRawHeadersIn: TStrings;
  end;
```
  1. In the response class, modernise the data members like so, and add a destructor.
```
  TApacheTwoResponse = class(TWebResponse)
  private
    FStringVariables: TStrings;
    FIntegerVariables: TList<Integer>;
    FDateVariables: TList<TDateTime>;
  public
    destructor Destroy; override;
```
  1. Replace the old FieldByName() method with...
```
function TApacheTwoRequest.GetFieldByName( const Name: AnsiString): AnsiString;
begin
result := apr_table_get( FRequest_rec^.headers_in^, pansichar(AnsiString(Name)));
if SameText( UTF8ToString( Name), 'SERVER_SOFTWARE') and (result = '') then
    result := GetStringVariable( ServerVers_Idx)
  else if SameText( UTF8ToString( Name), 'HANDLER') and (result = '') then
    result := GetStringVariable( Handler_Idx)
end;

function CompFunc( P: Pointer; PC: PAnsiChar; PC2: PAnsiChar): Integer; cdecl;
var
  as1, as2  : utf8string;
  Key, Value: string;
begin
if assigned( P) and (TObject( P) is TApacheTwoRequest) then
    begin
    if assigned( PC) then
        as1 := PC
      else
        as1 := '';
    Key := UTF8ToString( as1);
    if assigned( PC2) then
        as2 := PC2
      else
        as2 := '';
    Value := UTF8ToString( as2);
    if (Key = '') or (Value = '') or (as2[1] = #0) then
        result := 0
      else
        begin
        result := 1;
        TApacheTwoRequest(P).FRawHeaders.Add( Key + '=' + Value)
        end
    end
  else
    result := 0;
end;

function TApacheTwoRequest.GetRawHeadersIn: TStrings;
var
  valist: TBytes;
begin
if not assigned( FRawHeaders) then
  begin
  FRawHeaders := TStringList.Create;
  SetLength( vaList, 50);
  FillChar( vaList[0], Length(vaList), #0);
  try
    apr_table_vdo( @CompFunc, self, FRequest_rec^.headers_in^, @valist[0])
  except on E: Exception do
      begin
      if not assigned( FDummy) then
        FDummy := TStringList.Create;
      result := FDummy;
      FDummy.Clear;
      result.Add( E.ClassName + ': ' + E.Message);
      exit
      end
    end
  end;
result := FRawHeaders
end;


```
  1. Replace of the start of GetStringVariable(), before the big case statement with ...
```
function TApacheTwoRequest.GetStringVariable( Index: Integer): AnsiString;
const
  MaxChunkSize = 10240;

  function GetURI: string;
  var
    P1: integer;
  begin
  result := FRequest_rec^.the_request;
   // value like 'GET /hello/sdf?dfs=er HTTP/1.1'
   P1 := Pos( ' ', result);
   if P1 > 0 then
     Delete( result, 1, P1);
   P1 := Pos( ' ', result);
   if P1 > 0 then
     Delete( result, P1, MaxInt)
  end;

var
  len, BytesRead, Size: Integer;
  p, buf: pansichar;
  value: AnsiString;
  P2: integer;
begin
  case Index of
```
  1. Selectors 0 to 3 remain unchanged. I think Embarcadero is in error with selctor 4. My code for 4 is ...
```
    4: begin  // PathInfo
       value := FRequest_rec^.path_info;
       if value = '' then
         begin
         value := GetURI;
         P2 := Pos( '?', value);
         if P2 > 0 then
           Delete( value, P2, MaxInt)
         end
       end;
```
  1. Selectors 8 to 22 are ...
```
    8: value := apr_table_get (FRequest_rec^.headers_in^, 'Accept'); // Accept
    9: value := apr_table_get( FRequest_rec^.headers_in^, 'From');   // From
    10: value := FRequest_rec^.hostname;                             // Host
    12: value := apr_table_get(FRequest_rec^.headers_in^, 'Referer'); // Referer
    13: value := apr_table_get(FRequest_rec^.headers_in^, 'User-Agent'); // UserAgent
    14: value := FRequest_rec^.content_encoding;     // ContentEncoding
    15: value := apr_table_get(FRequest_rec^.headers_in^, 'Content-Type');  // ContentType
    16: value := apr_table_get(FRequest_rec^.headers_in^, 'Content-Length');
    17: value := ''; // ContentVersion
    18: value := ''; // DerivedFrom
    19: value := '';      // Expires
    20: value := apr_table_get(FRequest_rec^.headers_in^, 'Title');  // Title
    21: value := FRequest_rec^.connection^.client_ip; // RemoteAddr
    22: value := FRequest_rec^.connection^.remote_host; // RemoteHost
```
  1. 23, 24 and 25 are unchanged. 26 onwards are ..
```
    26: value := apr_table_get(FRequest_rec^.headers_in^, 'Connection'); // Connection
    27: value := apr_table_get(FRequest_rec^.headers_in^, 'Cookie');   // Cookie
    28: value := ''; // Authorization
    ServerName_Idx: Value := ap_get_server_name( FRequest_rec^);
    ServerVers_Idx: Value := ap_get_server_banner;
    Handler_Idx   : Value := FRequest_rec^.handler;
```
  1. Add this bit
```
destructor TApacheTwoRequest.Destroy;
begin
FRawHeaders.Free;
FDummy.Free;
inherited
end;
```
  1. Replace WriteString() with ...
```
function TApacheRequest.WriteString(const AString: AnsiString): Boolean;
begin
  Result := true;
  if Astring <> '' then
    Result := ap_rputs(PAnsiChar(Astring), FRequest_rec) = length(Astring)
end;
```
  1. Something is not right with TApacheTwoRequest.WriteHeaders(). Let me get back to you on what needs to change here.
  1. Change Version := '1.0' to Version := '1.1'
  1. Add ..
```
destructor TApacheTwoResponse.Destroy;
begin
FStringVariables.Free;
FIntegerVariables.Free;
FDateVariables.Free;
inherited
end;
```
  1. Change ...
```
function TApacheTwoResponse.GetDateVariable( Index: Integer): TDateTime;
begin
if assigned( FDateVariables) and (Index >= 0) and (Index < FDateVariables.Count) then
    result := FDateVariables[ Index]
  else
    result := 0.0
end;

function TApacheTwoResponse.GetIntegerVariable( Index: Integer): integer;
begin
if assigned( FIntegerVariables) and (Index >= 0) and (Index < FIntegerVariables.Count) then
    result := FIntegerVariables[ Index]
  else
    result := -1
end;

function TApacheTwoResponse.GetStringVariable( Index: Integer): AnsiString;
begin
if assigned( FStringVariables) and (Index >= 0) and (Index < FStringVariables.Count) then
    result := UTF8Encode( FStringVariables[ Index])
  else
    result := ''
end;
```
  1. Also ...
```
procedure TApacheTwoResponse.SetDateVariable( Index: Integer; const Value: TDateTime);
begin
if Index < 0 then exit;
if not assigned( FDateVariables) then
  FDateVariables := TList<TDateTime>.Create;
while FDateVariables.Count <= Index do
  FDateVariables.Add( 0.0);
FDateVariables[ Index] := Value
end;

procedure TApacheTwoResponse.SetIntegerVariable( Index: Integer; Value: Integer);
begin
if Index < 0 then exit;
if not assigned( FIntegerVariables) then
  FIntegerVariables := TList<integer>.Create;
while FIntegerVariables.Count <= Index do
  FIntegerVariables.Add( -1);
FIntegerVariables[ Index] := Value
end;

procedure TApacheTwoResponse.SetStringVariable( Index: Integer; const Value: AnsiString);
begin
if Index < 0 then exit;
if not assigned( FStringVariables) then
  FStringVariables := TStringList.Create;
while FStringVariables.Count <= Index do
  FStringVariables.Add( '');
FStringVariables[ Index] := Value
end;
```
  1. Inside SendResponse, change ...
```
  procedure AddHeaderItem( const Key, Value: AnsiString); overload;
  begin
  if (Key <> '') and (Value <> '') then
    with TApacheTwoRequest(FHTTPRequest) do
    apr_table_set( FRequest_rec.headers_out^, PAnsichar( Key), PAnsichar( Value))
  end;

  procedure AddCustomHeaders;
  var
    i: integer;
    Name: string;
  begin
  for i := 0 to FCustomHeaders.Count - 1 do
    begin
    Name := FCustomHeaders.Names[I];
    addHeaderItem(Name, FCustomHeaders.values[Name]);
    end
  end;

  procedure AddHeaderTimeItem( const Key: ansistring; Datum: TDateTime);
  var
    s: string;
  begin
  if Datum > 0.0 then
    begin
    // Eg. Datum = Sunday 25-Dec-2011 9:15pm
    s := FormatDateTime( sDateFormat + ' "GMT"', Datum);
    //  s = '"%s", 25 "%s" 2011 21:15:00 "GMT"'
    s := Format( s, [DayOfWeekStr( Datum), MonthStr( Datum)]);
    //  s = '"Sunday", 25 "December" 2011 21:15:00 "GMT"'
    {$WARN IMPLICIT_STRING_CAST OFF}
    AddHeaderItem( Key, s)
   {$WARN IMPLICIT_STRING_CAST ON}
    end
  end;
```
  1. The body of SendResponse becomes ...
```
begin
  if HTTPRequest.ProtocolVersion <> '' then
  begin
    if StatusCode > 0 then
      ServerMsg := Format('%d %s', [StatusCode, ReasonString]);

    AddHeaderItem(AnsiString('Allow'), Allow); {do not localize}

    for I := 0 to Cookies.Count - 1 do
    begin
      if (Cookies[I].HeaderValue <> '') then
        with TApacheTwoRequest(FHTTPRequest) do
          apr_table_add(FRequest_rec.headers_out^, pansichar('Set-Cookie'), {do not localize}
           PansiChar(AnsiString(Cookies[I].HeaderValue)));
    end;

    AddHeaderItem(AnsiString('Derived-From'), DerivedFrom); {do not localize}

    AddHeaderTimeItem( 'Expires'      , Expires);
    AddHeaderTimeItem( 'Last-Modified', LastModified);

    AddHeaderItem(AnsiString('Title'), Title);  {do not localize}
    AddHeaderItem(AnsiString('WWW-Authenticate'), WWWAuthenticate); {do not localize}
    AddCustomHeaders;
    AddHeaderItem(AnsiString('Content-Version'), ContentVersion); {do not localize}

    if ContentType <> '' then
      TApacheTwoRequest(FHTTPRequest).FRequest_rec.content_type := apr_pstrdup(TApacheTwoRequest(FHTTPRequest).FRequest_rec.pool^, pansichar(AnsiString(ContentType)));

    if ContentEncoding <> '' then
      TApacheTwoRequest(FHTTPRequest).FRequest_rec.content_encoding := apr_pstrdup(TApacheTwoRequest(FHTTPRequest).FRequest_rec.pool^, pansichar(AnsiString(ContentEncoding)));

    if (RawContent <> '') or (ContentStream <> nil) then
      AddHeaderItem('Content-Length', IntToStr(ContentLength));  {do not localize}

    HTTPRequest.WriteHeaders(StatusCode, AnsiString(ServerMsg), '');
  end;

  if ContentStream = nil then
    HTTPRequest.WriteString(RawContent)
  else if ContentStream <> nil then
  begin
    SendStream(ContentStream);
    ContentStream := nil; // Drop the stream
  end;

  FSent := True;
end;
```
  1. And finally change SendRedirect to
```
procedure TApacheTwoResponse.SendRedirect(const URI: AnsiString);
begin
  with TApacheTwoRequest(FHTTPRequest) do
  begin
    apr_table_set(FRequest_rec.headers_out^, 'Location', pansichar(URI)); {do not localize}
    FStatusCode := HTTP_MOVED_TEMPORARILY;
  end;
  FSent := False;
end;
```