GML:
```
/// Point order is:
///   P1 = top-left
///   P2 = top-right
///   P3 = bottom-left
///   P4 = bottom-right
/// 
/// @param sprite
/// @param image
/// @param x1
/// @param y1
/// @param x2
/// @param y2
/// @param x3
/// @param y3
/// @param x4
/// @param y4
/// @param [color=white]
/// @param [alpha=1]

function DrawSpriteHomographic(_sprite, _image, _x1, _y1, _x2, _y2, _x3, _y3, _x4, _y4, _color = c_white, _alpha = 1)
{
    static _vertexFormat = (function()
    {
        vertex_format_begin();
        vertex_format_add_position_3d();
        vertex_format_add_colour();
        vertex_format_add_texcoord();
        return vertex_format_end();
    })();
    
    static _vBuff = vertex_create_buffer();

    //Use cross products to figure out necessary perspective correction
    var _x14 = _x1 - _x4;
    var _y14 = _y1 - _y4;
    var _x23 = _x2 - _x3;
    var _y23 = _y2 - _y3;
    
    var _cross_23_14 = _x23*_y14 - _y23*_x14;
    if (_cross_23_14 != 0)
    {
        var _x43 = _x3 - _x4;
        var _y43 = _y3 - _y4;
        
        var _cross_23_43 = (_x23 * _y43 - _y23 * _x43) / _cross_23_14;
        if (_cross_23_43 != 0)
        {
            var _cross_14_43 = (_x14 * _y43 - _y14 * _x43) / _cross_23_14;
            if (_cross_14_43 != 0)
            {
                //Unpack UV data
                var _uvs      = sprite_get_uvs(_sprite, _image);
                var _uvLeft   = _uvs[0];
                var _uvTop    = _uvs[1];
                var _uvRight  = _uvs[2];
                var _uvBottom = _uvs[3];
                
                //Build out the vertex buffer
                //We use the z-coord of the position attribute as the w-component in the shader
                //You can think of the w-component in this situation as a "keystone" factor
                vertex_begin(_vBuff, _vertexFormat);
                vertex_position_3d(_vBuff, _x1, _y1,   _cross_23_43); vertex_colour(_vBuff, _color, _alpha); vertex_texcoord(_vBuff, _uvLeft,  _uvTop   );
                vertex_position_3d(_vBuff, _x2, _y2,   _cross_14_43); vertex_colour(_vBuff, _color, _alpha); vertex_texcoord(_vBuff, _uvRight, _uvTop   );
                vertex_position_3d(_vBuff, _x3, _y3, 1-_cross_14_43); vertex_colour(_vBuff, _color, _alpha); vertex_texcoord(_vBuff, _uvLeft,  _uvBottom);
                vertex_position_3d(_vBuff, _x4, _y4, 1-_cross_23_43); vertex_colour(_vBuff, _color, _alpha); vertex_texcoord(_vBuff, _uvRight, _uvBottom);
                vertex_end(_vBuff);
                
                //Render out as a triangle strip so we do less work GML-side
                shader_set(__shdDrawSpriteHomographic);
                vertex_submit(_vBuff, pr_trianglestrip, sprite_get_texture(_sprite, _image));
                shader_reset();
            }
        }
    }
}
```
Vertex shader:
```
attribute vec3 in_Position;
attribute vec4 in_Colour;
attribute vec2 in_TextureCoord;

varying vec2 v_vTexcoord;
varying vec4 v_vColour;

void main()
{
    //Calculate the world-space position separately...
    vec4 wsPos = gm_Matrices[MATRIX_WORLD]*vec4(in_Position.xy, 0.0, 1.0);
    
    //...so we can pass the "z" value from the input position as the w-component
    //This tricks the GPU into doing perspective correction on texture coordinates for us
    //We multiply up by the w-component to ensure the xyz positions are orthographically unchanged after texture perspective correction
    gl_Position = gm_Matrices[MATRIX_PROJECTION]*gm_Matrices[MATRIX_VIEW]*vec4(wsPos.xyz*in_Position.z, in_Position.z); 
    
    v_vColour   = in_Colour;
    v_vTexcoord = in_TextureCoord;
}
```
Frag shader:
```
varying vec2 v_vTexcoord;
varying vec4 v_vColour;

void main()
{
    gl_FragColor = v_vColour*texture2D(gm_BaseTexture, v_vTexcoord);
}
```
