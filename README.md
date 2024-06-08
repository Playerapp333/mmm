local s = {} do
	for _, v in pairs(game:GetChildren()) do
		if pcall(function() return v.ClassName end) and v.ClassName ~= '' and game:GetService(v.ClassName) then
			s[v.ClassName] = v
		end
	end
end
local senv = {}
local screenGui = Instance.new('ScreenGui')
screenGui.Name = 'MwsToolkit'
screenGui.ResetOnSpawn = false

local function makeGuiObject(className)
	local guiObject = Instance.new(className)
	
	guiObject.BackgroundColor3 = Color3.new(0, 0, 0)
	guiObject.BackgroundTransparency = 0.75
	guiObject.BorderSizePixel = 0
	guiObject.ClipsDescendants = false
	
	if guiObject:IsA('TextButton')
	or guiObject:IsA('TextLabel')
	or guiObject:IsA('TextBox') then
		guiObject.Font = Enum.Font.Gotham
		guiObject.LineHeight = 16 / 14
		guiObject.TextColor3 = Color3.new(255, 255, 255)
		guiObject.TextSize = 14
		guiObject.TextStrokeColor3 = Color3.new(0, 0, 0)
		guiObject.TextStrokeTransparency = 0.75
		
		if guiObject:IsA('TextLabel') then
			guiObject.BackgroundTransparency = 1
		elseif guiObject:IsA('TextBox') then
			guiObject.ClearTextOnFocus = false
		end
	elseif guiObject:IsA('ScrollingFrame') then
		guiObject.HorizontalScrollBarInset = Enum.ScrollBarInset.None
		guiObject.ScrollBarImageColor3 = Color3.new(1, 1, 1)
		guiObject.ScrollBarThickness = 4
		guiObject.ScrollingDirection = Enum.ScrollingDirection.Y
		guiObject.VerticalScrollBarInset = Enum.ScrollBarInset.None
		guiObject.Active = true
		guiObject.ClipsDescendants = true
	end

	s.Players.LocalPlayer.CharacterAdded:Connect(function()
		local bc = guiObject.BackgroundColor3
		guiObject:GetPropertyChangedSignal('BackgroundColor3'):Wait()
		guiObject.BackgroundColor3 = bc
	end)
	
	return guiObject
end

local class do
	local weakKeys = {__mode = 'k'}

	local classes = setmetatable({}, weakKeys)
	local instances = setmetatable({}, weakKeys)

	local classMeta
	local instanceMeta

	local globalPrototype
	local globalClassPrototype
	local globalInstancePrototype

	local deepCopy do
		function deepCopy(tbl, seen)
			if type(tbl) ~= 'table' then return tbl end
			if type(seen) ~= 'table' then seen = {} end
			if seen[tbl] then return seen[tbl] end
			
			local result = {}
			seen[tbl] = result
			
			for k, v in pairs(tbl) do
				result[k] = deepCopy(v, seen)
			end
			
			return result
		end
	end

	local immutableView do
		local thingies = {}
		local seens = {}
		local immutableViewMt = {}
		
		function immutableViewMt:__index(key)
			local value = thingies[self]
			
			if type(value) == 'table' then
				return immutableView(value, seens[self])
			end
			
			return value
		end
		
		function immutableViewMt:__newindex(key, value)
			error('Immutable table', 2)
		end
		
		function immutableViewMt:__len()
			return #thingies[self]
		end
		
		function immutableViewMt:__pairs()
			return pairs(thingies[self])
		end
		
		function immutableViewMt:__ipairs()
			return ipairs(thingies[self])
		end
		
		immutableViewMt.__metatable = '<immutable table>'
		
		function immutableView(tbl, seen)
			if type(tbl) ~= 'table' then return tbl end
			if type(seen) ~= 'table' then seen = {} end
			if seen[tbl] then return seen[tbl] end
			
			local proxy = setmetatable({}, immutableViewMt)
			thingies[proxy] = tbl
			seens[proxy] = seen
			seen[tbl] = proxy
			
			return proxy
		end
	end


	classMeta = {} do
		function classMeta:__index(key)
			local classData = classes[self]
			
			if key == '__Metatable' then
				if classData.metatableLocked then
					return immutableView(classData.metatable)
				end
				
				return classData.metatable
			elseif key == '__Preconstruct' then
				return classData.preconstructor
			elseif key == '__Construct' then
				return classData.constructor
			elseif key == '__SuperArgs' then
				return classData.superArgs
			elseif key == '__Class' then
				return self
			elseif key == '__Super' then
				return classData.superclass
			elseif key == '__ClassName' then
				return classData.name
			elseif key == '__ClassLib' then
				return class
			elseif globalClassPrototype[key] then
				return globalClassPrototype[key]
			elseif globalPrototype[key] then
				return globalPrototype[key]
			elseif key:sub(1, 2) == '__' then
				error('Cannot index field beginning with __ (reserved)', 2)
			end
			
			if classData.metatable and classData.metatable.__index then
				return classData.metatable.__index(self, key)
			end
			
			if classData.superclass then
				return classData.superclass[key]
			end
		end
		
		function classMeta:__newindex(key, value)
			local classData = classes[self]
			
			if key == '__Metatable' then
				if not value or type(value) == 'table' then
					if classData.metatableLocked then
						error('__Metatable is locked and cannot be changed', 2)
					end
					
					classData.metatable = value
				else
					error('__Metatable can only be set to a table or nil', 2)
				end
			elseif key == '__Preconstruct' then
				if value ~= nil and type(value) ~= 'function' then
					error('__Preconstruct can only be set to a function or nil', 2)
				end
				
				classData.preconstructor = value
			elseif key == '__Construct' then
				if value ~= nil and type(value) ~= 'function' then
					error('__Construct can only be set to a function or nil', 2)
				end
				
				classData.constructor = value
			elseif key == '__SuperArgs' then
				if value ~= nil and type(value) ~= 'function' then
					error('__SuperArgs can only be set to a function or nil', 2)
				end
				
				classData.superArgs = value
			elseif key == '__Class' then
				error('Cannot set __Class field of class', 2)
			elseif key == '__Super' then
				error('Cannot set __Super field of class', 2)
			elseif key == '__ClassName' then
				classData.name = value
			elseif key == '__ClassLib' then
				error('Cannot set __ClassLib field of class', 2)
			elseif globalClassPrototype[key] then
				error('Cannot override global class prototype', 2)
			elseif globalPrototype[key] then
				error('Cannot override global prototype', 2)
			elseif key:sub(1, 2) == '__' then
				error('Cannot index field beginning with __ (reserved)', 2)
			else
				if classData.metatable and classData.metatable.__newindex then
					return classData.metatable.__newindex(self, key, value)
				end
				
				rawset(self, key, value)
			end
		end
		
		function classMeta:__call(...)
			local classData = classes[self]
			
			local instance = {}
			local thisInstanceMeta = classData.instanceMeta
			
			if not thisInstanceMeta then
				thisInstanceMeta = {}
				
				local stack = {classData}
				
				while stack[#stack].superclass do
					stack[#stack + 1] = classes[stack[#stack].superclass]
				end
				
				for i = #stack, 1, -1 do
					local classData = stack[i]
					local newLock = not classData.metatableLocked
					classData.metatableLocked = true
					
					if classData.metatable then
						if newLock then
							classData.metatable = deepCopy(classData.metatable)
						end
						
						for k, v in pairs(classData.metatable) do
							thisInstanceMeta[k] = v
						end
					end
				end
				
				for k, v in pairs(instanceMeta) do
					thisInstanceMeta[k] = v
				end
				
				classData.instanceMeta = thisInstanceMeta
			end
			
			local finalInstance = setmetatable(instance, thisInstanceMeta)
			instances[finalInstance] = self
			
			if classData.preconstructor then
				classData.preconstructor(finalInstance, ...)
			end
			
			local stack = {classData}
			local argsStack = {{...}}
			
			while stack[#stack].superclass do
				local classData = stack[#stack]
				local superArgs = argsStack[#stack]
				
				if classData.superArgs then
					superArgs = {classData.superArgs(finalInstance, unpack(superArgs))}
				end
				
				local superclassData = classes[stack[#stack].superclass]
				stack[#stack + 1] = superclassData
				argsStack[#stack] = superArgs
				
				if superclassData.preconstructor then
					superclassData.preconstructor(finalInstance, unpack(superArgs))
				end
			end
			
			while #stack > 0 do
				local classData = stack[#stack]
				local args = argsStack[#stack]
				
				if classData.constructor then
					classData.constructor(finalInstance, unpack(args))
				end
				
				stack[#stack] = nil
			end
			
			return finalInstance
		end
		
		function classMeta:__tostring()
			return self.__ClassName
		end
		
		classMeta.__metatable = '<Class>'
	end

	instanceMeta = {} do
		function instanceMeta:__index(key)
			local instanceClass = instances[self]
			local classData = classes[instanceClass]
			
			if key == '__Metatable' then
				error('Cannot index __Metatable field of class through instance', 2)
			elseif key == '__Preconstruct' then
				error('Cannot index __Preconstruct method of class through instance', 2)
			elseif key == '__Construct' then
				error('Cannot index __Construct method of class through instance', 2)
			elseif key == '__SuperArgs' then
				error('Cannot index __SuperArgs method of class through instance', 2)
			elseif key == '__Class' then
				return instanceClass
			elseif key == '__Super' then
				return classData.superclass
			elseif key == '__ClassName' then
				return classData.name
			elseif key == '__ClassLib' then
				return class
			elseif globalInstancePrototype[key] then
				return globalInstancePrototype[key]
			elseif globalPrototype[key] then
				return globalPrototype[key]
			elseif key:sub(1, 2) == '__' then
				error('Cannot index field beginning with __ (reserved)', 2)
			end
			
			return instanceClass[key]
		end
		
		function instanceMeta:__newindex(key, value)
			if key == '__Metatable' then
				error('Cannot index __Metatable field of class through instance', 2)
			elseif key == '__Prefonstruct' then
				error('Cannot index __Preconstruct method of class through instance', 2)
			elseif key == '__Construct' then
				error('Cannot index __Construct method of class through instance', 2)
			elseif key == '__SuperArgs' then
				error('Cannot index __SuperArgs method of class through instance', 2)
			elseif key == '__Class' then
				error('Cannot set __Class field of class instance', 2)
			elseif key == '__Super' then
				error('Cannot set __Super field of class instance', 2)
			elseif key == '__ClassName' then
				error('Cannot set __ClassName field of class instance', 2)
			elseif key == '__ClassLib' then
				error('Cannot set __ClassLib field of class instance', 2)
			elseif globalInstancePrototype[key] then
				error('Cannot override global instance prototype', 2)
			elseif globalPrototype[key] then
				error('Cannot override global prototype', 2)
			elseif key:sub(1, 2) == '__' then
				error('Cannot index field beginning with __ (reserved)', 2)
			end
			
			rawset(self, key, value)
		end
		
		function instanceMeta:__tostring()
			local classData = classes[instances[self]]
			
			if classData.metatable and classData.metatable.__tostring then
				return classData.metatable.__tostring()
			end
			
			return classData.name .. '()'
		end
		
		instanceMeta.__metatable = '<ClassInstance>'
	end

	globalPrototype = {} do
		function globalPrototype:__IsA(class)
			if not classes[class] and type(class) ~= 'string' then
				error('Must call __IsA method with a class or class name', 2)
			end
			
			local current = self
			
			while current ~= nil do
				if current == class or current.__ClassName == class then return true end
				
				current = current.__Super
			end
			
			return false
		end
		
		function globalPrototype:__Subclass(name)
			return class(name, self.__Class)
		end
	end

	globalClassPrototype = {}
	globalInstancePrototype = {}

	class = setmetatable({
		IsClass = function(self, arg)
			return classes[arg] ~= nil
		end,
		IsInstance = function(self, arg)
			return instances[arg] ~= nil
		end,
		IsClassOrInstance = function(self, arg)
			return self:IsClass(arg) or self:IsInstance(arg)
		end,
		__InternalState__CAUTION_DO_NOT_USE_YOU_DO_NOT_NEED_THIS__ = {
			classes = classes,
			instances = instances,
			
			classMeta = classMeta,
			instanceMeta = instanceMeta,
			
			globalPrototype = globalPrototype,
			globalClassPrototype = globalClassPrototype,
			globalInstancePrototype = globalInstancePrototype
		}
	}, {
		__call = function(self, name, superclass)
			if type(name) ~= 'string' then
				error('Class must have a name', 2)
			end
			
			if superclass ~= nil and not classes[superclass] then
				error('Superclass is not a class', 2)
			end
			
			local class = {}
			local classData = {
				name = name,
				superclass = superclass,
				metatable = nil,
				metatableLocked = false,
				instanceMeta = nil,
				preconstructor = nil,
				constructor = nil,
				superArgs = nil
			}
			
			classes[class] = classData
			
			return setmetatable(class, classMeta)
		end
	})
end

local function makeDraggable(dragButton, dragObject)
	if typeof(dragButton) ~= 'Instance' or not dragButton:IsA('GuiButton') then
		error('Argument #1 must be GuiButton', 2)
	elseif typeof(dragObject) ~= 'Instance' or not dragObject:IsA('GuiObject') then
		error('Argument #2 must be GuiObject')
	end
	
	local dragOffset
	
	local onClick = dragButton.MouseButton1Down:Connect(function()
		dragOffset = dragObject.AbsolutePosition + dragObject.AbsoluteSize * dragObject.AnchorPoint - s.UserInputService:GetMouseLocation()
	end)
	
	local onDrag = s.UserInputService.InputChanged:Connect(function(input)
		if input.UserInputType ~= Enum.UserInputType.MouseMovement then return end
		
		if dragOffset then
			local tl, br = s.GuiService:GetGuiInset()
			local guiSpace = workspace.CurrentCamera.ViewportSize - tl - br
			local mousePos = s.UserInputService:GetMouseLocation()
			local xScale = (mousePos.X + dragOffset.X) / guiSpace.X
			local yScale = math.max((mousePos.Y + dragOffset.Y) / guiSpace.Y, 0)
			
			dragObject.Position = UDim2.new(xScale, 0, yScale, 0)
		end
	end)
	
	local onRelease = s.UserInputService.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseButton1 then
			dragOffset = nil
		end
	end)
	
	return {
		IsDragging = function() return dragOffset ~= nil end,
		Disconnect = function()
			onClick:Disconnect()
			onDrag:Disconnect()
			onRelease:Disconnect()
		end
	}
end

local Window = class('Window') do
	function Window:__Construct(title, parent)
		self.titlebar = makeGuiObject('TextButton')
		self.titlebar.Text = ''
		self.titlebar.BackgroundTransparency = 0.5
		self.titlebar.AutoButtonColor = false
		self.titlebar.Size = UDim2.new(0, 200, 0, 20)
		self.titlebar.AnchorPoint = Vector2.new(0.5, 0)
		self.titlebar.Position = UDim2.new(0.5, 0, 0.3, 0)
		
		makeDraggable(self.titlebar, self.titlebar)
		
		self.closeButton = makeGuiObject('TextButton')
		self.closeButton.Text = 'x'
		self.closeButton.BackgroundColor3 = Color3.new(0.9, 0.1, 0.1)
		self.closeButton.BackgroundTransparency = 0
		self.closeButton.Size = UDim2.new(0, 40, 0, 20)
		self.closeButton.AnchorPoint = Vector2.new(1, 0)
		self.closeButton.Position = UDim2.new(1, 0, 0, 0)
		self.closeButton.Parent = self.titlebar
		
		self.closeButton.MouseButton1Click:Connect(function()
			self:Close()
		end)
		
		self.collapseButton = makeGuiObject('TextButton')
		self.collapseButton.Text = ''
		self.collapseButton.Size = UDim2.new(0, 14, 0, 14)
		self.collapseButton.AnchorPoint = Vector2.new(0, 0.5)
		self.collapseButton.Position = UDim2.new(0, 3, 0.5)
		self.collapseButton.Parent = self.titlebar
		
		self.collapseIndicator = makeGuiObject('Frame')
		self.collapseIndicator.BackgroundColor3 = Color3.new(1, 1, 1)
		self.collapseIndicator.BackgroundTransparency = 0
		self.collapseIndicator.Size = UDim2.new(1, -6, 1, -6)
		self.collapseIndicator.AnchorPoint = Vector2.new(0.5, 0.5)
		self.collapseIndicator.Position = UDim2.new(0.5, 0, 0.5, 0)
		self.collapseIndicator.Parent = self.collapseButton
		
		self.windowTitle = makeGuiObject('TextLabel')
		self.windowTitle.Position = UDim2.new(0, 24, 0, 1)
		self.windowTitle.Size = UDim2.new(1, -64, 1, -1)
		self.windowTitle.TextXAlignment = Enum.TextXAlignment.Left
		self.windowTitle.Parent = self.titlebar
		
		self.content = makeGuiObject('Frame')
		self.content.Size = UDim2.new(1, 0, 0, 200)
		self.content.Position = UDim2.new(0, 0, 1, 0)
		self.content.Parent = self.titlebar
		
		self.rootContent = self.content
		self.collapseButton.MouseButton1Click:Connect(function()
			self.rootContent.Visible = not self.rootContent.Visible
			self.collapseIndicator.Visible = self.rootContent.Visible
		end)
		
		self:Title(title)
		self:Parent(parent)
	end
	
	function Window:Parent(parent)
		if parent ~= nil and typeof(parent) ~= 'Instance' then error('Parent must be Instance or nil', 2) end
		
		self.titlebar.Parent = parent
	end
	
	function Window:Title(title)
		self.windowTitle.Text = title
	end
	
	function Window:Close()
		self.titlebar:Destroy()
	end
	
	function Window:Width(width)
		if width == nil then return self.titlebar.Size.X.Offset end
		
		self.titlebar.Size = UDim2.new(0, width, 0, 20)
	end
	
	function Window:Height(height)
		if height == nil then return self.rootContent.Size.Y.Offset end
		
		self.rootContent.Size = UDim2.new(1, 0, 0, height)
	end
	
	function Window:IsClosed()
		return self.titlebar.Parent == nil
	end
end

local ScrollingWindow = Window:__Subclass('ScrollingWindow') do
	function ScrollingWindow:__Construct()
		self.contentContainer = self.content
		self.content = nil
		
		self.contentContainerPadding = Instance.new('UIPadding')
		self.contentContainerPadding.PaddingLeft = UDim.new(0, 4)
		self.contentContainerPadding.PaddingRight = UDim.new(0, 4)
		self.contentContainerPadding.Parent = self.contentContainer
		
		self.content = makeGuiObject('ScrollingFrame')
		self.content.BackgroundTransparency = 1
		self.content.Size = UDim2.new(1, 0, 1, 0)
		self.content.CanvasSize = UDim2.new(1, 0, 0, self.content.AbsoluteSize.Y)
		self.content.Parent = self.contentContainer
		
		self.contentPadding = Instance.new('UIPadding')
		self.contentPadding.PaddingTop = UDim.new(0, 8)
		self.contentPadding.PaddingBottom = UDim.new(0, 8)
		self.contentPadding.PaddingLeft = UDim.new(0, 8)
		self.contentPadding.PaddingRight = UDim.new(0, self.content.ScrollBarThickness + 12)
		self.contentPadding.Parent = self.content
	end
	
	function Window:ScrollHeight(height)
		if height == nil then return self.content.CanvasSize.Y.Offset end
		
		self.content.CanvasSize = UDim2.new(1, 0, 0, height)
	end
end

local MenuWindow = ScrollingWindow:__Subclass('MenuWindow') do
	function MenuWindow:__Construct()
		self.contentPadding.PaddingTop = UDim.new(0, 4)
		self.contentPadding.PaddingBottom = UDim.new(0, 4)
		
		self.contentList = Instance.new('UIListLayout')
		self.contentList.Padding = UDim.new(0, 4)
		self.contentList.SortOrder = Enum.SortOrder.LayoutOrder
		self.contentList.Parent = self.content
	end
	
	function MenuWindow:ResizeContent()
		self.content.CanvasSize = UDim2.new(1, 0, 0, self.contentList.AbsoluteContentSize.Y + self.contentPadding.PaddingTop.Offset + self.contentPadding.PaddingBottom.Offset)
	end
	
	function MenuWindow:MakeBtn(text, handler)
		local button = makeGuiObject('TextButton')
		button.Text = text
		button.Size = UDim2.new(1, 0, 0, 20)
		button.LayoutOrder = #self.content:GetChildren()
		button.Parent = self.content
		self:ResizeContent()
		
		if handler then
			button.MouseButton1Click:Connect(handler)
		end
		
		return button
	end
	
	function MenuWindow:MakeLabel(text)
		local label = makeGuiObject('TextLabel')
		label.Text = text
		label.Size = UDim2.new(1, 0, 0, 20)
		label.LayoutOrder = #self.content:GetChildren()
		label.Parent = self.content
		self:ResizeContent()
		
		return label
	end
	
	function MenuWindow:MakeObject(className)
		local object = makeGuiObject(className)
		object.Size = UDim2.new(1, 0, 0, 20)
		object.LayoutOrder = #self.content:GetChildren()
		object.Parent = self.content
		self:ResizeContent()
		
		return object
	end
	
	function MenuWindow:Padding(top, bottom)
		if top ~= nil then self.contentPadding.PaddingTop = UDim.new(0, top) end
		if bottom ~= nil then self.contentPadding.PaddingBottom = UDim.new(0, bottom) end
	end
end

local func_tpToBuildZone
local func_sellScrap
local func_perfectResize
local func_perfectPlace
local func_weldParts
local func_mergeItems
local func_detachPart
local func_decimateCreation
local func_weldHinge
local func_strongGrab
local func_whitelist
local func_tptool
local func_FiB
local func_Humancar
local func_delete
local func_fastdupe
local func_fling
local func_abandon
local func_goodbye
local func_ac
local func_glow
local func_swelds
local func_tt
local func_sk

local MwsUtils = {} do
	function MwsUtils:GetRemote(name)
		return s.ReplicatedStorage:WaitForChild('Remotes'):WaitForChild(name, 10)
	end
	
	function MwsUtils:ClearBuildZone()
		self:GetRemote('LoadSave'):InvokeServer('ClearBuildZone')
	end
	
	function MwsUtils:GetBuildZones()
		return workspace:WaitForChild('BuildZones')
	end
	
	function MwsUtils:IsBuildZone(model)
		return model.Parent == self:GetBuildZones()
	end
	
	function MwsUtils:GetBuildZoneOwner(buildZone)
		return buildZone:WaitForChild('ZoneInfo'):WaitForChild('OwnerPlayer').Value
	end
	
	function MwsUtils:GetMyBuildZone()
		for _, buildZone in ipairs(self:GetBuildZones():GetChildren()) do
			if self:GetBuildZoneOwner(buildZone) == s.Players.LocalPlayer then
				return buildZone
			end
		end
	end
	
	function MwsUtils:GetParts()
		return workspace:FindFirstChild('Parts')
	end
	
	function MwsUtils:GetItems()
		return workspace:FindFirstChild('Items')
	end
	
	function MwsUtils:GetWelds()
		return workspace:FindFirstChild('Welds')
	end
	
	function MwsUtils:GetTrees()
		return workspace:FindFirstChild('Trees')
	end
	
	function MwsUtils:Weld(part1, part2, part1ToWeld, part2ToWeld, weldLength)
		self:GetRemote('ClientMakeWeldJoint'):FireServer(part1, part2, part1ToWeld, part2ToWeld, weldLength)
	end
	
	function MwsUtils:RemoveWeld(weld)
		self:GetRemote('ClientCut'):FireServer(weld, Vector3.new(0, 0, 0), Vector3.new(0, 1, 0))
	end
	
	function MwsUtils:IsWeldable(part)
		return part:IsDescendantOf(self:GetParts()) or part:IsDescendantOf(self:GetItems()) or (part.Name == 'WoodSection' and part:IsDescendantOf(self:GetTrees())) or (part.Name == 'Main' and self:IsBuildZone(part.Parent))
	end
	
	function MwsUtils:GetPartWelds(part)
		local connected = part:GetConnectedParts(true)
		local welds = self:GetWelds()
		local out = {}
		
		for _, conn in ipairs(connected) do
			if conn:IsDescendantOf(welds) and (conn.A.Part1 == part or conn.B.Part1 == part) then
				out[#out + 1] = conn
			end
		end
		
		return out
	end
	
	function MwsUtils:GetItemModel(part)
		if part:IsDescendantOf(self:GetItems()) then
			local model = part
			
			while model.Parent ~= self:GetItems() do
				model = model.Parent
			end
			
			return model
		end
	end
	
	function MwsUtils:GetCreationWelds(part)
		local parts = self:GetParts()
		local items = self:GetItems()
		local welds = self:GetWelds()
		
		local connected = {[part] = true}
		local connecteds = 1
		
		while true do
			local oldConnecteds = connecteds
			local toAdd = {}
			
			for part, _ in pairs(connected) do
				local connectedParts = part:GetConnectedParts(true)
				
				if part:IsDescendantOf(items) then
					local model = self:GetItemModel(part)
					
					for _, desc in ipairs(model:GetDescendants()) do
						if desc:IsA('BasePart') then
							connectedParts[#connectedParts + 1] = desc
						end
					end
				end
				
				for _, connectedPart in ipairs(connectedParts) do
					if not toAdd[connectedPart] and not connected[connectedPart] and (self:IsWeldable(connectedPart) or connectedPart:IsDescendantOf(self:GetWelds())) and not connectedPart.Anchored then
						connecteds = connecteds + 1
						toAdd[connectedPart] = true
					end
				end
			end
			
			for k, v in pairs(toAdd) do
				connected[k] = v
			end
			
			if oldConnecteds == connecteds then break end
		end
		
		local connectedList = {}
		
		for part, _ in pairs(connected) do
			connected[part] = nil
			connectedList[#connectedList + 1] = part
		end
		
		local out = {}
		
		for _, conn in ipairs(connectedList) do
			if conn:IsDescendantOf(welds) then
				out[#out + 1] = conn
			end
		end
		
		return out
	end
	
	function MwsUtils:CutPart(part, pos, axis)
		self:GetRemote('ClientCut'):FireServer(part, pos, axis)
		
		local added = {}
		local conn = self:GetParts().ChildAdded:Connect(function(newPart)
			if newPart:IsA('BasePart') then
				added[#added + 1] = newPart
			end
		end)
		
		part.AncestryChanged:Wait()
		s.RunService.Heartbeat:Wait()
		conn:Disconnect()
		
		local filter = part.Size - part.Size * axis
		local newPartPos = part.CFrame.Position
		
		table.sort(added, function(a, b)
			return (a.CFrame.Position - newPartPos).Magnitude > (b.CFrame.Position - newPartPos).Magnitude
		end)
		
		local secondClosest
		local closest
		
		for _, part in ipairs(added) do
			if not closest or (part.CFrame.Position - newPartPos).Magnitude <= (closest.CFrame.Position - newPartPos).Magnitude then
				if (filter.X < 0.001 or math.abs(part.Size.X - filter.X) < 0.001)
				and (filter.Y < 0.001 or math.abs(part.Size.Y - filter.Y) < 0.001)
				and (filter.Z < 0.001 or math.abs(part.Size.Z - filter.Z) < 0.001) then
					secondClosest, closest = closest, part
				end
			end
		end
		
		return closest, secondClosest
	end
	
	function MwsUtils:IsItem(item)
		return item.Parent == self:GetItems()
	end
	
	function MwsUtils:IsHingeItem(item)
		local isItem = self:IsItem(item)
		local h1 = isItem and item:FindFirstChild('H1')
		local h2 = isItem and item:FindFirstChild('H2')
		local attachment0 = h1 and h1:FindFirstChild('Attachment0')
		local attachment1 = h2 and h2:FindFirstChild('Attachment1')
		local hc = attachment0 and attachment1 and h1:FindFirstChild('HingeConstraint')
		local hinge = hc and hc.Attachment0 == attachment0 and hc.Attachment1 == attachment1
		
		return hinge
	end
	
	function MwsUtils:GetHingePieces(item)
		if not self:IsHingeItem(item) then return nil end
		
		local isItem = self:IsItem(item)
		local h1 = isItem and item:FindFirstChild('H1')
		local h2 = isItem and item:FindFirstChild('H2')
		local attachment0 = h1 and h1:FindFirstChild('Attachment0')
		local attachment1 = h2 and h2:FindFirstChild('Attachment1')
		local hc = attachment0 and attachment1 and h1:FindFirstChildWhichIsA('HingeConstraint')
		
		return {
			h1 = h1,
			h2 = h2,
			attachment0 = attachment0,
			attachment1 = attachment1,
			constraint = hc
		}
	end
	
	function MwsUtils:IsMotorItem(item)
		local isItem = self:IsItem(item)
		local shaft = isItem and item:FindFirstChild('Shaft')
		local constraint = shaft and item:FindFirstChildWhichIsA('HingeConstraint')
		local attachment0 = constraint and constraint.Attachment0
		local attachment1 = constraint and constraint.Attachment1
		local shaftBase = attachment1 and attachment1.Parent
		local motor = shaftBase and attachment0 and attachment1
		
		return motor
	end
	
	function MwsUtils:GetMotorPieces(item)
		if not self:IsMotorItem(item) then return nil end
		
		local isItem = self:IsItem(item)
		local shaft = isItem and item:FindFirstChild('Shaft')
		local constraint = shaft and item:FindFirstChildWhichIsA('HingeConstraint')
		local attachment0 = constraint and constraint.Attachment0
		local attachment1 = constraint and constraint.Attachment1
		local shaftBase = attachment1 and attachment1.Parent
		
		return {
			shaft = shaft,
			shaftBase = shaftBase,
			constraint = constraint,
			attachment0 = attachment0,
			attachment1 = attachment1
		}
	end
	
	function MwsUtils:GetObjectManipulationScript()
		return s.Players.LocalPlayer.PlayerGui:WaitForChild('ObjectManipulation')
	end
	
	local interactionLocks = 0
	local bodyPosition
	local bodyGyro
	
	function MwsUtils:FindBodyMovers()
		if bodyPosition and bodyGyro then return end
		
		local objectManipulation = self:GetObjectManipulationScript()
		
		bodyPosition = objectManipulation:FindFirstChild('BodyPosition')
		bodyGyro = objectManipulation:FindFirstChild('BodyGyro')
		
		if not bodyPosition or not bodyGyro then
			bodyPosition = nil
			bodyGyro = nil
			
			-- heck
			
			local toSearch = self:GetParts():GetChildren()
			
			for _, item in ipairs(self:GetItems():GetChildren()) do
				for _, desc in ipairs(item:GetDescendants()) do
					if desc:IsA('BasePart') then
						toSearch[#toSearch + 1] = desc
					end
				end
			end
			
			for _, part in ipairs(toSearch) do
				bodyPosition = part:FindFirstChild('BodyPosition')
				bodyGyro = part:FindFirstChild('BodyGyro')
				
				if bodyPosition and bodyGyro then
					break
				else
					bodyPosition = nil
					bodyGyro = nil
				end
			end
			
			if not bodyPosition or not bodyGyro then
				bodyPosition = nil
				bodyGyro = nil
				
				-- HECK
				if getsenv then

					local success, senv = pcall(function() getsenv(objectManipulation) end)
					
					if success then
                    if senv == nil then senv = {} end
						for k, v in pairs(senv) do
							if v:IsA('BodyPosition') then
								bodyPosition = v
                                wait()
							elseif v:IsA('BodyGyro') then
								bodyGyro = v
                                wait()
							end
						end
					end
				end
				
				if not bodyPosition or not bodyGyro then
					bodyPosition = nil
					bodyGyro = nil
					
					-- FUCK
					
					if getnilinstances then
						local nilInstances = getnilinstances()
						
						for _, instance in ipairs(nilInstances) do
							if instance:IsA('BodyPosition') and instance.Name == 'BodyPosition' then
								bodyPosition = instance
							elseif instance:IsA('BodyGyro') and instance.Name == 'BodyGyro' then
								bodyGyro = instance
							end
							
							if bodyPosition and bodyGyro then break end
						end
						
						if not bodyPosition or not bodyGyro then
							bodyPosition = nil
							bodyGyro = nil
							
							-- WHAT
						end
					end
				end
			end
		end
	end
	
	function MwsUtils:LockInteraction()
		if interactionLocks == 0 then
			local objectManipulation = self:GetObjectManipulationScript()
			
			objectManipulation.Disabled = true
			
			for _, child in ipairs(objectManipulation:GetChildren()) do
				if child:IsA('SelectionBox') then
					child.Adornee = nil
				end
			end
			
			self:FindBodyMovers()
		end
		
		interactionLocks = interactionLocks + 1
	end
	
	local function isDestroyed(x)
		if x.Parent then return false end
		local _, result = pcall(function() x.Parent = x end)
		return result:match('locked') and true or false
	end
	
	function MwsUtils:UnlockInteraction()
		interactionLocks = interactionLocks - 1
		
		if interactionLocks == 0 then
			local objectManipulation = self:GetObjectManipulationScript()
			
			if not bodyPosition or not bodyGyro or isDestroyed(bodyPosition) or isDestroyed(bodyGyro) then
				local _bodyPosition = bodyPosition
				local _bodyGyro = bodyGyro
				
				bodyPosition = Instance.new('BodyPosition')
				bodyGyro = Instance.new('BodyGyro')
				
				bodyPosition.D = 200
				bodyPosition.MaxForce = Vector3.new(1, 1, 1) * 30000
				bodyPosition.P = 3000
				
				bodyGyro.D = 100
				bodyGyro.MaxTorque = Vector3.new(1, 1, 1) * 10000
				bodyGyro.P = 2500
				
				if _bodyPosition and _bodyGyro then
					bodyPosition.D = _bodyPosition.D
					bodyPosition.MaxForce = _bodyPosition.MaxForce
					bodyPosition.P = _bodyPosition.P
					
					bodyGyro.D = _bodyGyro.D
					bodyGyro.MaxTorque = _bodyGyro.MaxTorque
					bodyGyro.P = _bodyGyro.P
					
					local bodyPosition = bodyPosition
					local bodyGyro = bodyGyro
					
					_bodyPosition.Changed:Connect(function(key)
						bodyPosition[key] = _bodyPosition[key]
					end)
					
					_bodyGyro.Changed:Connect(function(key)
						bodyGyro[key] = _bodyGyro[key]
					end)
				end
			end
			
			bodyPosition.Parent = objectManipulation
			bodyGyro.Parent = objectManipulation
			
			objectManipulation.Disabled = false
		end
	end
	
	local function characterAdded(character)
		local bp = bodyPosition
		local bg = bodyGyro

		bodyPosition = nil
		bodyPosition = nil
		
		if bp and bg then
			bp.Parent = nil
			bg.Parent = nil
		end
		
		MwsUtils:LockInteraction()
		
		if bp and bg then
			if bodyPosition and bodyGyro then
				local _bodyPosition = bodyPosition
				local _bodyGyro = bodyGyro
				
				bodyPosition = nil
				bodyGyro = nil
				
				_bodyPosition:Destroy()
				_bodyGyro:Destroy()
			end
			
			bodyPosition = bp
			bodyGyro = bg
		end
		
		MwsUtils:UnlockInteraction()
	end
	
	if s.Players.LocalPlayer.Character then characterAdded(s.Players.LocalPlayer.Character) end
	s.Players.LocalPlayer.CharacterAdded:Connect(characterAdded)
	
	if getrawmetatable and setreadonly and getnamecallmethod then
		local game_mt = getrawmetatable(game)
		local game_namecall = game_mt.__namecall
		
		setreadonly(game_mt, false)
		
		game_mt.__namecall = function(self, ...)
			local method = getnamecallmethod()
			
			if getnamecallmethod() == 'Destroy' and (self == bodyPosition or self == bodyGyro) then
				self.Parent = nil
				return
			end
			
			return game_namecall(self, ...)
		end
		
		setreadonly(game_mt, true)
	end
	
	function MwsUtils:GetObjectManipulationBodyMovers()
		self:FindBodyMovers()
		
		return {
			BodyPosition = bodyPosition,
			BodyGyro = bodyGyro
		}
	end
end
local function func_tt()
local mouse = game:GetService('Players').LocalPlayer:GetMouse()
local inv = {}
tool = Instance.new("Tool")
tool.RequiresHandle = false
tool.Name = "toggle transparency"
local function getpart()
local part = mouse.Target
if not part:IsDescendantOf(game.Workspace.Trees) and not part:IsDescendantOf(game.Workspace.Map) and not part:IsDescendantOf(game.Workspace.BuildZones) and not part:IsDescendantOf(game.Workspace.TrashCans) then return part end
end
tool.Activated:connect(function()
local part = getpart()
if not table.find(inv, part) then
if part:IsDescendantOf(workspace.Parts) then
table.insert(inv, part)
part.Transparency = 0.7
else
for i, v in ipairs (part.Parent:GetChildren()) do
if v.ClassName == 'Part' or v.ClassName == 'UnionOperation' then
table.insert(inv, v)
v.Transparency = 0.7
end
end
end
else if table.find(inv, part) and part:IsDescendantOf(workspace.Parts) then
table.remove(inv, table.find(inv, part))
part.Transparency = 0
else if table.find(inv, part) and part:IsDescendantOf(workspace.Items) then
for a, d in ipairs(part.Parent:GetChildren()) do
if d.ClassName == 'Part' or d.ClassName == 'UnionOperation' then
table.remove(inv, table.find(inv, d))
d.Transparency = 0
end end
end
end
end
end)
tool.Parent = game.Players.LocalPlayer.Backpack
end
local function func_FiB()
if _G.FibTeardown then pcall(_G.FibTeardown) end

local lighting = game:GetService('Lighting')

local function update()
    lighting.EnvironmentDiffuseScale = 1
    lighting.EnvironmentSpecularScale = 1
    lighting.Brightness = 2

    lighting.OutdoorAmbient = Color3.new()

    lighting.ClockTime = 15

    lighting.FogEnd = 999999
end

local conn = game:GetService('RunService').RenderStepped:Connect(update)

local colorCorrection = Instance.new('ColorCorrectionEffect')
colorCorrection.TintColor = Color3.fromRGB(255, 200, 150)
colorCorrection.Parent = lighting

local bloom = Instance.new('BloomEffect')
bloom.Intensity = 2
bloom.Size = 32
bloom.Threshold = 2
bloom.Parent = lighting

local function lightAdded(light)
    light.Shadows = true
end

local descConn = workspace.DescendantAdded:Connect(function(desc)
    if desc:IsA('Light') then lightAdded(desc) end
end)

for _, desc in ipairs(workspace:GetDescendants()) do
    if desc:IsA('Light') then lightAdded(desc) end
end

_G.FibTeardown = function()
    conn:Disconnect()
    colorCorrection:Destroy()
    bloom:Destroy()
    descConn:Disconnect()
end
end
local function func_tptool()
 mouse = game.Players.LocalPlayer:GetMouse()
tool = Instance.new("Tool")
tool.RequiresHandle = false
tool.Name = "TP"
tool.Activated:connect(function()
local pos = mouse.Hit+Vector3.new(0,2.5,0)
pos = CFrame.new(pos.X,pos.Y,pos.Z)
game.Players.LocalPlayer.Character.HumanoidRootPart.CFrame = pos
end)
tool.Parent = game.Players.LocalPlayer.Backpack
end
local function func_tpToBuildZone()
	local localPlayer = s.Players.LocalPlayer
	
	local buildZone = MwsUtils:GetMyBuildZone()
	
	if not buildZone then
		local closest
		
		for _, buildZone in ipairs(MwsUtils:GetBuildZones():GetChildren()) do
			if (not closest or (closest.Main.Position - localPlayer.Character.PrimaryPart.Position).Magnitude > (buildZone.Main.Position - localPlayer.Character.PrimaryPart.Position).Magnitude) and not MwsUtils:GetBuildZoneOwner(buildZone) then
				closest = buildZone
			end
		end
		
		buildZone = closest
	end
	
	if buildZone then
		localPlayer.Character:SetPrimaryPartCFrame(buildZone.Main.CFrame * CFrame.new(0, 6, 0))
	end
end

local function func_sellScrap()
	MwsUtils:ClearBuildZone()
end

local function func_whitelist()
    require(game.ReplicatedStorage.InteractionPermission).canInteract = function() return true end
end
local function startPartSelection(parent, callback, filter)
	local toReturn
	
	local localPlayer = s.Players.LocalPlayer
	local mouse = localPlayer:GetMouse()
	
	local selectionBox = Instance.new('SelectionBox', parent)
	selectionBox.Name = 'SelectingBox'
	selectionBox.Transparency = 0.25
	selectionBox.LineThickness = 0.05
	selectionBox.Color3 = Color3.new(0.75, 0.75, 0.75)
	
	local c1 = s.RunService.RenderStepped:Connect(function()
		local ray = workspace.CurrentCamera:ScreenPointToRay(mouse.X, mouse.Y)
		ray = Ray.new(ray.Origin, ray.Direction * 5000)
		local hit = workspace:FindPartOnRay(ray, localPlayer.Character, false, true)
		
		if hit and MwsUtils:IsWeldable(hit) and (not filter or filter(hit)) then
			selectionBox.Adornee = hit
		else
			selectionBox.Adornee = nil
		end
	end)
	
	local c2, c3
	
	do
		local downPart, x, y
		
		c2 = mouse.Button1Down:Connect(function()
			downPart = selectionBox.Adornee
			x, y = mouse.X, mouse.Y
		end)
		
		c3 = mouse.Button1Up:Connect(function()
			if selectionBox.Adornee == downPart and (downPart ~= nil or mouse.X == x and mouse.Y == y) then
				c1:Disconnect()
				c2:Disconnect()
				c3:Disconnect()
				selectionBox.Transparency = 0
				selectionBox.Color3 = Color3.new(4 / 255, 175 / 255, 236 / 255)
				callback(downPart, toReturn)
			end
			
			downPart, x, y = nil, nil, nil
		end)
	end
	
	toReturn = {
		DestroySelectionBox = function(self)
			selectionBox:Destroy()
		end,
		Cancel = function(self)
			c1:Disconnect()
			c2:Disconnect()
			c3:Disconnect()
			self:DestroySelectionBox()
			MwsUtils:UnlockInteraction()
		end,
		InProgress = function(self)
			return c1.Connected
		end,
		SelectionBox = selectionBox
	}
	
	MwsUtils:LockInteraction()
	
	return toReturn
end

local function numbersOnly(textBox)
	if typeof(textBox) ~= 'Instance' or not textBox:IsA('TextBox') then
		error('Argument #1 must be TextBox', 2)
	end
	
	local decimalsAllowed = false
	local negativesAllowed = false
	
	local connection = textBox.Changed:Connect(function(property)
		if property ~= 'Text' then return end
		
		local text = textBox.Text
		local cursorPos = textBox.CursorPosition
		local selStart = textBox.SelectionStart
		local periodPos = text:find('%.')

		-- fricking stupid calamari bug
		local len = text:len()
		
		for i = len, 1, -1 do
			local char = text:sub(i, i)
			
			if char:match('%d') == nil and (not decimalsAllowed or periodPos == -1 or i ~= periodPos) and (not negativesAllowed or i > 1 or char ~= '-') then
				text = text:sub(1, i - 1) .. text:sub(i + 1)
				
				if cursorPos > i then
					cursorPos = cursorPos - 1
				end
				
				if selStart > i then
					selStart = selStart - 1
				end
			end
		end
		
		textBox.Text = text
		textBox.CursorPosition = cursorPos
		textBox.SelectionStart = selStart
	end)
	
	return {
		SetDecimalsAllowed = function(setDecimalsAllowed)
			decimalsAllowed = setDecimalsAllowed
		end,
		SetNegativesAllowed = function(setNegativesAllowed)
			negativesAllowed = setNegativesAllowed
		end,
		Disconnect = function(self)
			connection:Disconnect()
		end,
		GetNumber = function(self, default)
			if tonumber(textBox.Text) == nil then
				return default or 0
			end
			
			return tonumber(textBox.Text)
		end
	}
end

local PerfectResizeWindow = Window:__Subclass('PerfectResizeWindow') do
	function PerfectResizeWindow:__SuperArgs(parent)
		return 'Perfect Resize', parent
	end
	
	function PerfectResizeWindow:__Construct()
		self.partSelection = makeGuiObject('TextButton')
		self.partSelection.Text = 'Click to select'
		self.partSelection.Size = UDim2.new(0, 150, 0, 20)
		self.partSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.partSelection.Position = UDim2.new(0.5, 0, 0, 8)
		self.partSelection.Parent = self.content
		
		self.currentSelection = nil
		self.currentSelecting = nil
		
		self.partSelection.MouseButton1Click:Connect(function()
			if self.resizing then return end
			
			if self.currentSelecting then
				self.currentSelecting:Cancel()
				self.currentSelecting = nil
				self.currentSelection = nil
				self:UpdateUI()
				
				return
			end
			
			self:StartPartSelection()
			self:UpdateUI()
		end)
		
		self.extraContainer = makeGuiObject('Frame')
		self.extraContainer.BackgroundTransparency = 1
		self.extraContainer.Size = UDim2.new(1, -8, 0, 68)
		self.extraContainer.AnchorPoint = Vector2.new(0.5, 0)
		self.extraContainer.Position = UDim2.new(0.5, 0, 0, 36)
		self.extraContainer.Parent = self.content
		
		self.axisLabelX = makeGuiObject('TextLabel')
		self.axisLabelX.Text = 'X'
		self.axisLabelX.Size = UDim2.new(1 / 3, 0, 0, 20)
		self.axisLabelX.Position = UDim2.new(0, 0, 0, 0)
		self.axisLabelX.Parent = self.extraContainer
		
		self.axisLabelY = makeGuiObject('TextLabel')
		self.axisLabelY.Text = 'Y'
		self.axisLabelY.Size = UDim2.new(1 / 3, 0, 0, 20)
		self.axisLabelY.Position = UDim2.new(1 / 3, 0, 0, 0)
		self.axisLabelY.Parent = self.extraContainer
		
		self.axisLabelZ = makeGuiObject('TextLabel')
		self.axisLabelZ.Text = 'Z'
		self.axisLabelZ.Size = UDim2.new(1 / 3, 0, 0, 20)
		self.axisLabelZ.Position = UDim2.new(2 / 3, 0, 0, 0)
		self.axisLabelZ.Parent = self.extraContainer
		
		local function createVisualizeThing(textBox, axis)
			local adornment
			
			textBox.Focused:Connect(function()
				adornment = Instance.new('CylinderHandleAdornment')
				adornment.Radius = 0.1
				adornment.Height = self.currentSelection.Size:Dot(axis) + 1
				adornment.Adornee = self.currentSelection
				adornment.CFrame = CFrame.new(Vector3.new(0, 0, 0), axis)
				adornment.Color3 = Color3.new(4 / 255, 175 / 255, 236 / 255)
				adornment.Parent = workspace.CurrentCamera
			end)
			
			textBox.FocusLost:Connect(function()
				if adornment then
					adornment:Destroy()
				end
			end)
		end
		
		self.textBoxX = makeGuiObject('TextBox')
		self.textBoxX.Text = ''
		self.textBoxX.Size = UDim2.new(1 / 3, -8, 0, 20)
		self.textBoxX.Position = UDim2.new(0, 4, 0, 20)
		self.textBoxX.Parent = self.extraContainer
		self.numbersOnlyX = numbersOnly(self.textBoxX)
		self.numbersOnlyX:SetDecimalsAllowed(true)
		createVisualizeThing(self.textBoxX, Vector3.new(1, 0, 0))
		
		self.textBoxY = makeGuiObject('TextBox')
		self.textBoxY.Text = ''
		self.textBoxY.Size = UDim2.new(1 / 3, -8, 0, 20)
		self.textBoxY.Position = UDim2.new(1 / 3, 4, 0, 20)
		self.textBoxY.Parent = self.extraContainer
		self.numbersOnlyY = numbersOnly(self.textBoxY)
		self.numbersOnlyY:SetDecimalsAllowed(true)
		createVisualizeThing(self.textBoxY, Vector3.new(0, 1, 0))
		
		self.textBoxZ = makeGuiObject('TextBox')
		self.textBoxZ.Text = ''
		self.textBoxZ.Size = UDim2.new(1 / 3, -8, 0, 20)
		self.textBoxZ.Position = UDim2.new(2 / 3, 4, 0, 20)
		self.textBoxZ.Parent = self.extraContainer
		self.numbersOnlyZ = numbersOnly(self.textBoxZ)
		self.numbersOnlyZ:SetDecimalsAllowed(true)
		createVisualizeThing(self.textBoxZ, Vector3.new(0, 0, 1))
		
		self.resizeButton = makeGuiObject('TextButton')
		self.resizeButton.Text = 'Resize'
		self.resizeButton.Size = UDim2.new(1, -8, 0, 20)
		self.resizeButton.Position = UDim2.new(0, 4, 0, 48)
		self.resizeButton.Parent = self.extraContainer
		
		self.resizing = false
		
		self.resizeButton.MouseButton1Click:Connect(function()
			if not self.resizing then self:StartResizing() end
		end)
		
		self:UpdateUI()
	end
	
	function PerfectResizeWindow:UpdateUI()
		if self.currentSelecting and self.currentSelecting:InProgress() then
			self.partSelection.Font = Enum.Font.GothamBold
			self.partSelection.Text = 'Click part to select'
		else
			self.partSelection.Font = Enum.Font.Gotham
			self.partSelection.Text = self.currentSelecting and 'Click to cancel' or 'Click to select'
		end
		
		self.extraContainer.Visible = self.currentSelection ~= nil
		
		if self.currentSelection then
			self:Height(112)
			
			self.textBoxX.TextEditable = not self.resizing
			self.textBoxY.TextEditable = not self.resizing
			self.textBoxZ.TextEditable = not self.resizing
			self.textBoxX.TextTransparency = self.resizing and 0.25 or 0
			self.textBoxY.TextTransparency = self.resizing and 0.25 or 0
			self.textBoxZ.TextTransparency = self.resizing and 0.25 or 0
			self.resizeButton.AutoButtonColor = not self.resizing
			self.resizeButton.Text = self.resizing and 'Resizing...' or 'Resize'
			self.resizeButton.TextTransparency = self.resizing and 0.25 or 0
			self.partSelection.AutoButtonColor = not self.resizing
			self.partSelection.TextTransparency = self.resizing and 0.25 or 0
		else
			self:Height(36)
		end
	end
	
	function PerfectResizeWindow:StartPartSelection()
		if self.currentSelecting then self.currentSelecting:DestroySelectionBox() end
		self.currentSelection = nil
		self.currentSelecting = startPartSelection(workspace.CurrentCamera, function(selected)
			self.currentSelection = selected
			
			if self.currentSelection then
				self.textBoxX.Text = string.format('%0.4f', self.currentSelection.Size.X)
				self.textBoxY.Text = string.format('%0.4f', self.currentSelection.Size.Y)
				self.textBoxZ.Text = string.format('%0.4f', self.currentSelection.Size.Z)
			end
			
			self:UpdateUI()
		end, function(part)
			return not (part:IsDescendantOf(MwsUtils:GetItems()) or part.Anchored)
		end)
	end
	
	function PerfectResizeWindow:Close()
		if self.resizing then return end
		
		if self.currentSelecting then
			self.currentSelecting:Cancel()
		end
		
		self.__Super.Close(self)
	end
	
	function PerfectResizeWindow:ResizeOnce(axis, targetLength)
		self.currentSelection = MwsUtils:CutPart(self.currentSelection, axis * (targetLength - (self.currentSelection.Size:Dot(axis) / 2)), axis)
	end
	
	function PerfectResizeWindow:StartResizing()
		if self.resizing then error('Already resizing', 2) end
		self.resizing = true
		self:UpdateUI()
		
		local targetX = self.numbersOnlyX:GetNumber()
		local targetY = self.numbersOnlyY:GetNumber()
		local targetZ = self.numbersOnlyZ:GetNumber()
		
		if self.currentSelection.Size.X - targetX > 0.0001 then
			self:ResizeOnce(Vector3.new(1, 0, 0), targetX)
		end
		
		self.currentSelecting.SelectionBox.Adornee = self.currentSelection
		
		if self.currentSelection.Size.Y - targetY > 0.0001 then
			self:ResizeOnce(Vector3.new(0, 1, 0), targetY)
		end
		
		self.currentSelecting.SelectionBox.Adornee = self.currentSelection
		
		if self.currentSelection.Size.Z - targetZ > 0.0001 then
			self:ResizeOnce(Vector3.new(0, 0, 1), targetZ)
		end
		
		self.currentSelecting.SelectionBox.Adornee = self.currentSelection
		self.textBoxX.Text = string.format('%0.4f', self.currentSelection.Size.X)
		self.textBoxY.Text = string.format('%0.4f', self.currentSelection.Size.Y)
		self.textBoxZ.Text = string.format('%0.4f', self.currentSelection.Size.Z)
		
		self.resizing = false
		self:UpdateUI()
	end
end

local perfectResizeWindow

local function func_perfectResize()
	if perfectResizeWindow and not perfectResizeWindow:IsClosed() then return end
	perfectResizeWindow = PerfectResizeWindow(screenGui)
end

local startSurfaceSelection do
	local normalids = {
		[Vector3.new(0, 1, 0)] = Enum.NormalId.Top,
		[Vector3.new(0, -1, 0)] = Enum.NormalId.Bottom,
		[Vector3.new(0, 0, 1)] = Enum.NormalId.Back,
		[Vector3.new(0, 0, -1)] = Enum.NormalId.Front,
		[Vector3.new(1, 0, 0)] = Enum.NormalId.Right,
		[Vector3.new(-1, 0, 0)] = Enum.NormalId.Left
	}

	local vectors = {}

	for k, v in pairs(normalids) do
		vectors[v] = k
	end

	for k, v in pairs(normalids) do
		normalids[v] = k
	end

	local function normalIdFromVector(vector)
		for v, nid in pairs(normalids) do
			if typeof(v) == 'Vector3' and v:FuzzyEq(vector, math.sqrt(2) / 2) then
				return nid
			end
		end
		
		return nil
	end

	function startSurfaceSelection(parent, callback, filterFunc)
		local toReturn
		
		local localPlayer = s.Players.LocalPlayer
		local mouse = localPlayer:GetMouse()
		
		local surfaceSelection = Instance.new('SurfaceSelection', parent)
		surfaceSelection.Name = 'SurfaceSelection'
		surfaceSelection.Transparency = 0.25
		surfaceSelection.Color3 = Color3.new(0.75, 0.75, 0.75)
		
		local c1 = s.RunService.RenderStepped:Connect(function()
			local ray = workspace.CurrentCamera:ScreenPointToRay(mouse.X, mouse.Y)
			ray = Ray.new(ray.Origin, ray.Direction * 5000)
			local hit, _, normal = workspace:FindPartOnRay(ray, localPlayer.Character, false, true)
			local normalId = hit and normalIdFromVector(hit.CFrame:VectorToObjectSpace(normal))
			
			if hit and MwsUtils:IsWeldable(hit) and normalId and (not filterFunc or filterFunc(hit, normalId, vectors[normalId])) then
				surfaceSelection.Adornee = hit
				surfaceSelection.TargetSurface = normalId
			else
				surfaceSelection.Adornee = nil
			end
		end)
		
		local c2, c3
		
		do
			local downPart, x, y
			
			c2 = mouse.Button1Down:Connect(function()
				downPart = surfaceSelection.Adornee
				x, y = mouse.X, mouse.Y
			end)
			
			c3 = mouse.Button1Up:Connect(function()
				if surfaceSelection.Adornee == downPart and (downPart ~= nil or mouse.X == x and mouse.Y == y) then
					c1:Disconnect()
					c2:Disconnect()
					c3:Disconnect()
					surfaceSelection.Transparency = 0
					surfaceSelection.Color3 = Color3.new(4 / 255, 175 / 255, 236 / 255)
					callback(downPart, vectors[surfaceSelection.TargetSurface], toReturn)
				end
				
				downPart, x, y = nil, nil, nil
			end)
		end
		
		toReturn = {
			DestroySelectionBox = function(self)
				surfaceSelection:Destroy()
			end,
			Cancel = function(self)
				c1:Disconnect()
				c2:Disconnect()
				c3:Disconnect()
				self:DestroySelectionBox()
				MwsUtils:UnlockInteraction()
			end,
			InProgress = function(self)
				return c1.Connected
			end,
			SelectionBox = surfaceSelection
		}
		
		MwsUtils:LockInteraction()
		
		return toReturn
	end
end

local function MwsGhost(root)
	local neatFolder = Instance.new('Folder')
	neatFolder.Name = 'MwsToolkitGhost'
	neatFolder.Parent = workspace.CurrentCamera
	local clones = {}
	local parts = {}
	
	local function makeEquivalent(part, root, offset)
		local clone = part:Clone()
		
		for _, child in ipairs(clone:GetChildren()) do
			if not child:IsA('SpecialMesh') then
				child:Remove()
			end
		end
		
		clones[part] = clone
		parts[clone] = part
		clone.Parent = neatFolder
		clone.Transparency = 1 - (1 - clone.Transparency) * 0.5
		clone.CanCollide = false
		clone.Massless = true
		
		if root then
			local weld = Instance.new('Weld')
			weld.Part0 = clone
			weld.Part1 = root
			weld.C0 = offset
			weld.Parent = clone
		end
		
		return clone
	end
	
	local rootClone = makeEquivalent(root)
	rootClone.Transparency = rootClone.Transparency / 2
	
	local function update()
		local conns = root:GetConnectedParts(true)
		
		local connsLookup = {}
		for _, v in pairs(conns) do connsLookup[v] = true end
		
		for _, conn in ipairs(conns) do
			if conn:IsDescendantOf(MwsUtils:GetItems()) then
				for _, part in ipairs(MwsUtils:GetItemModel(conn):GetDescendants()) do
					if part:IsA('BasePart') and MwsUtils:IsWeldable(conn) and not connsLookup[part] then
						conns[#conns + 1] = part
						connsLookup[part] = true
					end
				end
			end
		end
		
		for _, conn in ipairs(conns) do
			if not clones[conn] and MwsUtils:IsWeldable(conn) or conn.Parent == MwsUtils:GetWelds() then
				makeEquivalent(conn, rootClone, conn.CFrame:ToObjectSpace(root.CFrame))
			end
		end
		
		for k, part in ipairs(parts) do
			if not connsLookup[part] then
				clones[part]:Destroy()
				clones[part] = nil
				parts[k] = nil
			end
		end
	end
	
	local conn = game:GetService('RunService').RenderStepped:Connect(update)
	
	return {
		Cancel = function(self)
			conn:Disconnect()
			neatFolder:Destroy()
		end,
		UpdateCFrame = function(self, cframe)
			rootClone.CFrame = cframe
		end,
		InProgress = function(self)
			return conn.Connected
		end,
		GetFolder = function(self)
			return neatFolder
		end,
		GetRootClone = function(self)
			return rootClone
		end
	}
end

local isExploit = not s.RunService:IsStudio()

local PerfectPlaceWindow = Window:__Subclass('PerfectPlaceWindow') do
	function PerfectPlaceWindow:__SuperArgs(parent)
		return 'Perfect Place', parent
	end
	
	function PerfectPlaceWindow:__Construct()
		self.partSelection = makeGuiObject('TextButton')
		self.partSelection.Text = 'Click to select'
		self.partSelection.Size = UDim2.new(0, 150, 0, 20)
		self.partSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.partSelection.Position = UDim2.new(0.5, 0, 0, 8)
		self.partSelection.Parent = self.content
		
		self.fineTuning = false
		
		self.partSelection.MouseButton1Click:Connect(function()
			if self:InProgress() then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self:Start()
			self:UpdateUI()
		end)
		
		self.extraContainer = makeGuiObject('Frame')
		self.extraContainer.BackgroundTransparency = 1
		self.extraContainer.Size = UDim2.new(1, -8, 0, 76)
		self.extraContainer.AnchorPoint = Vector2.new(0.5, 0)
		self.extraContainer.Position = UDim2.new(0.5, 0, 0, 36)
		self.extraContainer.Parent = self.content
		
		self.moveSnapLabel = makeGuiObject('TextLabel')
		self.moveSnapLabel.Text = 'Move Snap'
		self.moveSnapLabel.TextXAlignment = Enum.TextXAlignment.Left
		self.moveSnapLabel.Size = UDim2.new(0.5, -8, 0, 20)
		self.moveSnapLabel.AnchorPoint = Vector2.new(0.5, 0)
		self.moveSnapLabel.Position = UDim2.new(0.25, 0, 0, 0)
		self.moveSnapLabel.Parent = self.extraContainer
		
		self.moveSnapBox = makeGuiObject('TextBox')
		self.moveSnapBox.Text = '0.5'
		self.moveSnapBox.Size = UDim2.new(0.5, -8, 0, 20)
		self.moveSnapBox.AnchorPoint = Vector2.new(0.5, 0)
		self.moveSnapBox.Position = UDim2.new(0.75, 0, 0, 0)
		self.moveSnapBox.Parent = self.extraContainer
		self.numbersOnlyMoveSnap = numbersOnly(self.moveSnapBox)
		self.numbersOnlyMoveSnap:SetDecimalsAllowed(true)
		
		self.rotateSnapLabel = makeGuiObject('TextLabel')
		self.rotateSnapLabel.Text = 'Rotate Snap'
		self.rotateSnapLabel.TextXAlignment = Enum.TextXAlignment.Left
		self.rotateSnapLabel.Size = UDim2.new(0.5, -8, 0, 20)
		self.rotateSnapLabel.AnchorPoint = Vector2.new(0.5, 0)
		self.rotateSnapLabel.Position = UDim2.new(0.25, 0, 0, 28)
		self.rotateSnapLabel.Parent = self.extraContainer
		
		self.rotateSnapBox = makeGuiObject('TextBox')
		self.rotateSnapBox.Text = '15'
		self.rotateSnapBox.Size = UDim2.new(0.5, -8, 0, 20)
		self.rotateSnapBox.AnchorPoint = Vector2.new(0.5, 0)
		self.rotateSnapBox.Position = UDim2.new(0.75, 0, 0, 28)
		self.rotateSnapBox.Parent = self.extraContainer
		self.numbersOnlyRotateSnap = numbersOnly(self.rotateSnapBox)
		self.numbersOnlyRotateSnap:SetDecimalsAllowed(true)
		
		self.toolButton = makeGuiObject('TextButton')
		self.toolButton.Text = 'Moving'
		self.toolButton.Size = UDim2.new(0.5, -8, 0, 20)
		self.toolButton.AnchorPoint = Vector2.new(0.5, 0)
		self.toolButton.Position = UDim2.new(0.25, 0, 0, 56)
		self.toolButton.Parent = self.extraContainer
		
		self.targetButton = makeGuiObject('TextButton')
		self.targetButton.Text = 'Everything'
		self.targetButton.Size = UDim2.new(0.5, -8, 0, 20)
		self.targetButton.AnchorPoint = Vector2.new(0.5, 0)
		self.targetButton.Position = UDim2.new(0.75, 0, 0, 56)
		self.targetButton.Parent = self.extraContainer
		
		self.weldButton = makeGuiObject('TextButton')
		self.weldButton.Text = 'Weld'
		self.weldButton.Size = UDim2.new(1, -8, 0, 20)
		self.weldButton.AnchorPoint = Vector2.new(0.5, 0)
		self.weldButton.Position = UDim2.new(0.5, 0, 0, 84)
		self.weldButton.Parent = self.extraContainer
		
		self:UpdateUI()
	end
	
	function PerfectPlaceWindow:UpdateUI()
		if self:InProgress() and not self:IsFineTuning() then
			self.partSelection.Font = Enum.Font.GothamBold
			self.partSelection.Text = 'Click surface to select'
		else
			self.partSelection.Font = Enum.Font.Gotham
			self.partSelection.Text = self:IsFineTuning() and 'Click to cancel' or 'Click to select'
		end
		
		self.extraContainer.Visible = self:IsFineTuning()
		
		self:Height(self:IsFineTuning() and 144 or 36)
	end
	
	function PerfectPlaceWindow:Start()
		self.cancels = {}
		
		local selecting = startSurfaceSelection(workspace.CurrentCamera, function(selected, normal)
			if not selected or not normal then self:Cancel() return end
			
			local selecting = startSurfaceSelection(workspace.CurrentCamera, function(selected2, normal2)
				self:Cancel()
				
				if not selected2 or not normal2 then return end
				
				self.cancels = {}
				
				self.sel1Part = selected
				self.sel1Normal = normal
				self.sel2Part = selected2
				self.sel2Normal = normal2
				
				self.cancels[1] = function()
					self.sel1Part = nil
					self.sel1Normal = nil
					self.sel2Part = nil
					self.sel2Normal = nil
				end
				
				self:StartFineTuning()
				self:UpdateUI()
			end, function(hit, normal)
				return hit ~= selected
			end)
		
			self.cancels[2] = function()
				selecting:Cancel()
			end
		end, function(part)
			return not part.Anchored
		end)
		
		self.cancels[1] = function()
			selecting:Cancel()
		end
	end
	
	function PerfectPlaceWindow:Cancel()
		for _, cancel in ipairs(self.cancels) do
			cancel()
		end
		
		self.cancels = nil
		self:UpdateUI()
	end
	
	function PerfectPlaceWindow:InProgress()
		return self.cancels ~= nil
	end
	
	function PerfectPlaceWindow:IsFineTuning()
		return self.fineTuning
	end
	
	function PerfectPlaceWindow:Close()
		if self:InProgress() then
			self:Cancel()
		end
		
		self.__Super.Close(self)
	end
	
	function PerfectPlaceWindow:StartFineTuning()
		self.fineTuning = true
		
		local sel1Part = self.sel1Part
		local sel1Normal = self.sel1Normal
		local sel2Part = self.sel2Part
		local sel2Normal = self.sel2Normal
		
		local ghost = MwsGhost(sel1Part)
		local ghostOffset
		local rootOffset
		local movingJustWeld = false
		
		local function resetPosition()
			ghostOffset = CFrame.new(sel1Normal * sel1Part.Size / 2, Vector3.new(0, 0, 0))
			rootOffset = CFrame.new(Vector3.new(0, 0, 0), sel2Normal * sel2Part.Size / 2) + sel2Normal * sel2Part.Size / 2
		end
		
		resetPosition()
		
		local weldPart = Instance.new('Part')
		Instance.new('SpecialMesh', weldPart).MeshType = Enum.MeshType.Sphere
		weldPart.Size = Vector3.new(0.2, 0.2, 0.2)
		weldPart.Parent = ghost:GetRootClone().Parent
		weldPart.Material = Enum.Material.SmoothPlastic
		weldPart.BrickColor = BrickColor.new('Institutional white')
		weldPart.Massless = true
		
		local weldA = Instance.new('Weld')
		weldA.Part0 = ghost:GetRootClone()
		weldA.Part1 = weldPart
		weldA.Parent = weldPart
		
		local weldB = Instance.new('Weld')
		weldB.Part0 = sel2Part
		weldB.Part1 = weldPart
		weldB.Parent = weldPart
		
		local moveHandlesX = Instance.new('Handles')
		moveHandlesX.Style = Enum.HandlesStyle.Movement
		moveHandlesX.Color3 = Color3.new(1, 0, 0)
		moveHandlesX.Faces = Faces.new(Enum.NormalId.Left, Enum.NormalId.Right)
		moveHandlesX.Adornee = weldPart
		moveHandlesX.Parent = isExploit and s.CoreGui or self.titlebar.Parent
		
		local moveHandlesY = Instance.new('Handles')
		moveHandlesY.Style = Enum.HandlesStyle.Movement
		moveHandlesY.Color3 = Color3.new(0, 1, 0)
		moveHandlesY.Faces = Faces.new(Enum.NormalId.Top, Enum.NormalId.Bottom)
		moveHandlesY.Adornee = weldPart
		moveHandlesY.Parent = isExploit and s.CoreGui or self.titlebar.Parent
		
		local moveHandlesZ = Instance.new('Handles')
		moveHandlesZ.Style = Enum.HandlesStyle.Movement
		moveHandlesZ.Color3 = Color3.new(0, 0, 1)
		moveHandlesZ.Faces = Faces.new(Enum.NormalId.Front, Enum.NormalId.Back)
		moveHandlesZ.Adornee = weldPart
		moveHandlesZ.Parent = isExploit and s.CoreGui or self.titlebar.Parent
		
		local function moveDraggable(moveHandles)
			local lastRoot
			local lastGhost
			local moveSnap
			
			moveHandles.MouseButton1Down:Connect(function()
				local adornee = moveHandlesX.Adornee
				
				lastRoot = rootOffset
				lastGhost = ghostOffset
				
				moveSnap = self.numbersOnlyMoveSnap:GetNumber()
			end)
			
			moveHandles.MouseDrag:Connect(function(face, distance)
				local adornee = moveHandlesX.Adornee
				local roundedDistance = moveSnap == 0 and distance or math.floor(math.abs(distance) / moveSnap) * moveSnap * math.sign(distance)
				local offset = CFrame.new(Vector3.FromNormalId(face) * roundedDistance)
				
				if adornee == weldPart then
					rootOffset = lastRoot * offset
				end
				
				if adornee ~= weldPart then
					ghostOffset = offset:Inverse() * lastGhost
				end
				
				if movingJustWeld then
					ghostOffset = lastGhost * offset
				end
			end)
			
			moveHandles.MouseButton1Up:Connect(function()
				lastRoot = nil
				lastGhost = nil
				moveSnap = nil
			end)
		end
		
		moveDraggable(moveHandlesX)
		moveDraggable(moveHandlesY)
		moveDraggable(moveHandlesZ)
		
		local arcHandlesX = Instance.new('ArcHandles')
		arcHandlesX.Color3 = Color3.new(1, 0, 0)
		arcHandlesX.Axes = Axes.new(Enum.Axis.X)
		arcHandlesX.Adornee = weldPart
		arcHandlesX.Visible = false
		arcHandlesX.Parent = isExploit and s.CoreGui or self.titlebar.Parent
		
		local arcHandlesY = Instance.new('ArcHandles')
		arcHandlesY.Color3 = Color3.new(0, 1, 0)
		arcHandlesY.Axes = Axes.new(Enum.Axis.Y)
		arcHandlesY.Adornee = weldPart
		arcHandlesY.Visible = false
		arcHandlesY.Parent = isExploit and s.CoreGui or self.titlebar.Parent
		
		local arcHandlesZ = Instance.new('ArcHandles')
		arcHandlesZ.Color3 = Color3.new(0, 0, 1)
		arcHandlesZ.Axes = Axes.new(Enum.Axis.Z)
		arcHandlesZ.Adornee = weldPart
		arcHandlesZ.Visible = false
		arcHandlesZ.Parent = isExploit and s.CoreGui or self.titlebar.Parent
		
		local function arcDraggable(arcHandles)
			local lastRoot
			local lastGhost
			local arcSnap
			
			arcHandles.MouseButton1Down:Connect(function()
				local adornee = arcHandlesX.Adornee
				
				lastRoot = rootOffset
				lastGhost = ghostOffset
				
				arcSnap = self.numbersOnlyRotateSnap:GetNumber()
			end)
			
			arcHandles.MouseDrag:Connect(function(axis, angle)
				angle = math.deg(angle)
				local adornee = moveHandlesX.Adornee
				local roundedAngle = arcSnap == 0 and angle or math.floor(math.abs(angle) / arcSnap) * arcSnap * math.sign(angle)
				local offset = CFrame.fromAxisAngle(Vector3.FromAxis(axis), math.rad(roundedAngle))
				
				if adornee == weldPart then
					rootOffset = lastRoot * offset
				end
				
				if adornee ~= weldPart then
					ghostOffset = offset:Inverse() * lastGhost
				end
				
				if movingJustWeld then
					ghostOffset = lastGhost * offset
				end
			end)
			
			arcHandles.MouseButton1Up:Connect(function()
				lastRoot = nil
				lastGhost = nil
				arcSnap = nil
			end)
		end
		
		arcDraggable(arcHandlesX)
		arcDraggable(arcHandlesY)
		arcDraggable(arcHandlesZ)
		
		local c1 = self.toolButton.MouseButton1Click:Connect(function()
			local rotating = arcHandlesX.Visible
			
			moveHandlesX.Visible = rotating
			moveHandlesY.Visible = rotating
			moveHandlesZ.Visible = rotating
			
			arcHandlesX.Visible = not rotating
			arcHandlesY.Visible = not rotating
			arcHandlesZ.Visible = not rotating
			
			self.toolButton.Text = rotating and 'Moving' or 'Rotating'
		end)
		
		local c2 = self.targetButton.MouseButton1Click:Connect(function()
			local adornee = moveHandlesX.Adornee
			local newAdornee = adornee == weldPart and ghost:GetRootClone() or weldPart
			
			if not movingJustWeld then
				if adornee ~= weldPart then
					movingJustWeld = true
					newAdornee = weldPart
				end
			else
				movingJustWeld = false
				newAdornee = weldPart
			end
			
			moveHandlesX.Adornee = newAdornee
			moveHandlesY.Adornee = newAdornee
			moveHandlesZ.Adornee = newAdornee
			
			arcHandlesX.Adornee = newAdornee
			arcHandlesY.Adornee = newAdornee
			arcHandlesZ.Adornee = newAdornee
			
			self.targetButton.Text = newAdornee == weldPart and (movingJustWeld and 'Weld' or 'Everything') or 'Part'
		end)
		
		local function update()
			weldA.C0 = ghostOffset
			weldB.C0 = rootOffset
		end
		
		update()
		
		local c3 = s.RunService.RenderStepped:Connect(update)
		
		local c4 = self.weldButton.MouseButton1Click:Connect(function()
			self:Cancel()
			MwsUtils:Weld(sel1Part, sel2Part, ghostOffset, rootOffset, 0)
		end)
		
		self.cancels[2] = function()
			moveHandlesX:Destroy()
			moveHandlesY:Destroy()
			moveHandlesZ:Destroy()
			arcHandlesX:Destroy()
			arcHandlesY:Destroy()
			arcHandlesZ:Destroy()
			weldPart:Destroy()
			ghost:Cancel()
			c1:Disconnect()
			c2:Disconnect()
			c3:Disconnect()
			c4:Disconnect()
			self.toolButton.Text = 'Moving'
			self.targetButton.Text = 'Everything'
			self.fineTuning = false
		end
	end
end

local perfectPlaceWindow

local function func_perfectPlace()
	if perfectPlaceWindow and not perfectPlaceWindow:IsClosed() then return end
	perfectPlaceWindow = PerfectPlaceWindow(screenGui)
end

local WeldPartsWindow = Window:__Subclass('WeldPartsWindow') do
	function WeldPartsWindow:__SuperArgs(parent)
		return 'Weld Parts', parent
	end
	
	function WeldPartsWindow:__Construct()
		self.itemSelection = makeGuiObject('TextButton')
		self.itemSelection.Text = 'Click to select'
		self.itemSelection.Size = UDim2.new(0, 150, 0, 20)
		self.itemSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.itemSelection.Position = UDim2.new(0.5, 0, 0, 8)
		self.itemSelection.Parent = self.content
		
		self.itemSelection.MouseButton1Click:Connect(function()
			if self:InProgress() then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self:Start()
			self:UpdateUI()
		end)
		
		self:UpdateUI()
	end
	
	function WeldPartsWindow:UpdateUI()
		if self:InProgress() then
			self.itemSelection.Font = Enum.Font.GothamBold
			self.itemSelection.Text = 'Click to select part ' .. (self.part2Select and '2' or '1')
		else
			self.itemSelection.Font = Enum.Font.Gotham
			self.itemSelection.Text = 'Click to select'
		end
		
		self:Height(36)
	end
	
	function WeldPartsWindow:Start()
		self.part1Select = startPartSelection(workspace.CurrentCamera, function(part1, data)
			self.part1 = part1
			
			if not self.part1 then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self.part2Select = startPartSelection(workspace.CurrentCamera, function(part2)
				self.part2 = part2
				
				if not self.part2 then
					self:Cancel()
					self:UpdateUI()
					
					return
				end
				
				local part1 = self.part1
				local part2 = self.part2
				
				if math.min(part1.Size.X, part1.Size.Y, part1.Size.Z) < 0.2 then
					if math.min(part2.Size.X, part2.Size.Y, part2.Size.Z) >= 0.2 then
						part1, part2 = part2, part1
					end
				end
				
				MwsUtils:Weld(part1, part2, CFrame.new(0, 0, 0), part2.CFrame:ToObjectSpace(part1.CFrame), 0)
				
				self:Cancel()
				self:UpdateUI()
			end, function(part)
				return part ~= part1
			end)
			
			self:UpdateUI()
		end)
		
		self:UpdateUI()
	end
	
	function WeldPartsWindow:Cancel()
		if self.part2Select then
			self.part1Select:Cancel()
			self.part2Select:Cancel()
		elseif self.part1Select then
			self.part1Select:Cancel()
		end
		
		self.part1Select = nil
		self.part2Select = nil
		self.part1 = nil
		self.part2 = nil
	end
	
	function WeldPartsWindow:InProgress()
		return self.part1Select ~= nil
	end
	
	function WeldPartsWindow:Close()
		if self:InProgress() then
			self:Cancel()
		end
		
		self.__Super.Close(self)
	end
end

local weldPartsWindow

local function func_weldParts()
	if weldPartsWindow and not weldPartsWindow:IsClosed() then return end
	weldPartsWindow = WeldPartsWindow(screenGui)
end

local startItemSelection = function(parent, callback, filter)
	local toReturn
	
	local localPlayer = s.Players.LocalPlayer
	local mouse = localPlayer:GetMouse()
	
	local selectionBoxPart = Instance.new('Part')
	
	local selectionBox = Instance.new('SelectionBox', parent)
	selectionBox.Name = 'SelectingBox'
	selectionBox.Transparency = 0.25
	selectionBox.LineThickness = 0.05
	selectionBox.Color3 = Color3.new(0.75, 0.75, 0.75)
	selectionBox.Visible = false
	selectionBox.Adornee = selectionBoxPart
	
	local hoverItem
	
	local c1 = s.RunService.RenderStepped:Connect(function()
		local ray = workspace.CurrentCamera:ScreenPointToRay(mouse.X, mouse.Y)
		ray = Ray.new(ray.Origin, ray.Direction * 5000)
		local hit = workspace:FindPartOnRay(ray, localPlayer.Character, false, true)
		hit = hit and MwsUtils:GetItemModel(hit)
		
		if hit and (not filter or filter(hit)) then
			hoverItem = hit
			local cframe, size = hit:GetBoundingBox()
			selectionBoxPart.Size = size
			selectionBoxPart.CFrame = cframe
			
			selectionBox.Visible = true
		else
			hoverItem = nil
			selectionBox.Visible = false
		end
	end)
	
	local c2, c3
	
	do
		local downItem, x, y
		
		c2 = mouse.Button1Down:Connect(function()
			downItem = hoverItem
			x, y = mouse.X, mouse.Y
		end)
		
		c3 = mouse.Button1Up:Connect(function()
			if hoverItem == downItem and (downItem ~= nil or mouse.X == x and mouse.Y == y) then
				c1:Disconnect()
				c2:Disconnect()
				c3:Disconnect()
				selectionBox.Transparency = 0
				selectionBox.Color3 = Color3.new(4 / 255, 175 / 255, 236 / 255)
				callback(downItem, toReturn)
			end
			
			downItem, x, y = nil, nil, nil
		end)
	end
	
	toReturn = {
		DestroySelectionBox = function(self)
			selectionBox:Destroy()
			selectionBoxPart:Destroy()
		end,
		Cancel = function(self)
			c1:Disconnect()
			c2:Disconnect()
			c3:Disconnect()
			self:DestroySelectionBox()
			MwsUtils:UnlockInteraction()
		end,
		InProgress = function(self)
			return c1.Connected
		end,
		SelectionBox = selectionBox
	}
	
	MwsUtils:LockInteraction()
	
	return toReturn
end

local MergeItemsWindow = Window:__Subclass('MergeItemsWindow') do
	function MergeItemsWindow:__SuperArgs(parent)
		return 'Merge Items', parent
	end
	
	function MergeItemsWindow:__Construct()
		self.itemSelection = makeGuiObject('TextButton')
		self.itemSelection.Text = 'Click to select'
		self.itemSelection.Size = UDim2.new(0, 150, 0, 20)
		self.itemSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.itemSelection.Position = UDim2.new(0.5, 0, 0, 8)
		self.itemSelection.Parent = self.content
		
		self.itemSelection.MouseButton1Click:Connect(function()
			if self.active then return end
			
			if self:InProgress() then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self:Start()
			self:UpdateUI()
		end)
		
		self:UpdateUI()
	end
	
	function MergeItemsWindow:UpdateUI()
		if self:InProgress() and not self.active then
			self.itemSelection.Font = Enum.Font.GothamBold
			self.itemSelection.Text = 'Click to select item ' .. (self.item2Select and '2' or '1')
		else
			self.itemSelection.Font = Enum.Font.Gotham
			self.itemSelection.Text = self.active and 'Merging...' or 'Click to select'
		end
		
		self.itemSelection.AutoButtonColor = not self.active
		self.itemSelection.TextTransparency = self.active and 0.25 or 0
		
		self:Height(36)
	end
	
	function MergeItemsWindow:Start()
		self.item1Select = startItemSelection(workspace.CurrentCamera, function(item1, data)
			self.item1 = item1
			
			if not self.item1 then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self.item2Select = startItemSelection(workspace.CurrentCamera, function(item2)
				self.item2 = item2
				
				if not self.item2 then
					self:Cancel()
					self:UpdateUI()
					
					return
				end
				
				local item1 = self.item1
				local item2 = self.item2
				
				self.active = true
				self:UpdateUI()
				
				for _, child in ipairs(item1:GetChildren()) do
					if child:IsA('BasePart') then
						local child2 = item2:FindFirstChild(child.Name)
						
						if not child2 and s.RunService:IsStudio() then warn(child2.Name) end
						
						wait(0.25)
						MwsUtils:Weld(child, child2, CFrame.new(), CFrame.new(), 0)
					end
				end
				self.active = false
				
				self:Cancel()
				self:UpdateUI()
			end, function(item)
				return item ~= item1 and item.Name == item1.Name
			end)
			
			self:UpdateUI()
		end)
		
		self:UpdateUI()
	end
	
	function MergeItemsWindow:Cancel()
		if self.item2Select then
			self.item1Select:Cancel()
			self.item2Select:Cancel()
		elseif self.item1Select then
			self.item1Select:Cancel()
		end
		
		self.item1Select = nil
		self.item2Select = nil
		self.item1 = nil
		self.item2 = nil
	end
	
	function MergeItemsWindow:InProgress()
		return self.item1Select ~= nil
	end
	
	function MergeItemsWindow:Close()
		if self:InProgress() then
			self:Cancel()
		end
		
		self.__Super.Close(self)
	end
end

local mergeItemsWindow

local function func_mergeItems()
	if mergeItemsWindow and not mergeItemsWindow:IsClosed() then return end
	mergeItemsWindow = MergeItemsWindow(screenGui)
end

local Widget = class('Widget') do
	function Widget:__Construct()
		self.frame = makeGuiObject('Frame')
		self.frame.BackgroundTransparency = 1
		self.frame.Size = UDim2.new(1, 0, 1, 0)
	end
	
	function Widget:Parent(parent)
		self.frame.Parent = parent
	end
end

local WidgetCheckbox = Widget:__Subclass('WidgetCheckbox') do
	function WidgetCheckbox:__Construct()
		self.btn = makeGuiObject('TextButton')
		self.btn.Text = ''
		self.btn.Size = UDim2.new(0, 14, 0, 14)
		self.btn.AnchorPoint = Vector2.new(0.5, 0.5)
		self.btn.Position = UDim2.new(0.5, 0, 0.5, 0)
		self.btn.Parent = self.frame
		
		self.indicator = makeGuiObject('Frame')
		self.indicator.BackgroundColor3 = Color3.new(1, 1, 1)
		self.indicator.BackgroundTransparency = 0
		self.indicator.Size = UDim2.new(1, -6, 1, -6)
		self.indicator.AnchorPoint = Vector2.new(0.5, 0.5)
		self.indicator.Position = UDim2.new(0.5, 0, 0.5, 0)
		self.indicator.Visible = false
		self.indicator.Parent = self.btn
		
		self._ChangedEvent = Instance.new('BindableEvent')
		self.Changed = self._ChangedEvent.Event
		
		self.checked = false
		
		self.btn.Activated:Connect(function()
			self:SetChecked(not self.checked)
		end)
	end
	
	function WidgetCheckbox:SetChecked(checked)
		local oldChecked = self.checked
		
		self.checked = checked
		self.indicator.Visible = checked
		
		if checked ~= oldChecked then
			self._ChangedEvent:Fire(checked)
		end
	end
end

local DupePartWindow = MenuWindow:__Subclass('DupePartWindow') do
	function DupePartWindow:__SuperArgs(parent)
		return 'Dupe Part', parent
	end
	
	function DupePartWindow:__Construct()
		self:Padding(8, 8)
		
		self.partSelectionContainer = self:MakeObject('Frame')
		self.partSelectionContainer.BackgroundTransparency = 1
		
		self.partSelection = makeGuiObject('TextButton')
		self.partSelection.Text = 'Click to select'
		self.partSelection.Size = UDim2.new(0, 150, 0, 20)
		self.partSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.partSelection.Position = UDim2.new(0.5, 0, 0, 0)
		self.partSelection.Parent = self.partSelectionContainer
		
		self.partSelection.MouseButton1Click:Connect(function()
			if self:InProgress() then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self:Start()
			self:UpdateUI()
		end)
		
		--[[self.multiplierFrame = self:MakeObject('Frame')
		self.multiplierFrame.BackgroundTransparency = 1
		
		self.multiplierLabel = makeGuiObject('TextLabel')
		self.multiplierLabel.Text = 'Multiplier'
		self.multiplierLabel.Size = UDim2.new(0.4, -4, 1, 0)
		self.multiplierLabel.TextXAlignment = Enum.TextXAlignment.Left
		self.multiplierLabel.Parent = self.multiplierFrame
		
		self.multiplierBox = makeGuiObject('TextBox')
		self.multiplierBox.Text = '0'
		self.multiplierBox.Size = UDim2.new(0.6, -4, 1, 0)
		self.multiplierBox.Position = UDim2.new(0.4, 4, 0, 0)
		self.multiplierBox.Parent = self.multiplierFrame
		self.numbersOnlyMultiplier = numbersOnly(self.multiplierBox)
		self.numbersOnlyMultiplier:SetDecimalsAllowed(true)]]
		
		self.weldContainer = self:MakeObject('Frame')
		self.weldContainer.BackgroundTransparency = 1
		
		self.weldLabel = makeGuiObject('TextLabel')
		self.weldLabel.Text = 'Weld to Build Zone'
		self.weldLabel.Size = UDim2.new(1, -22, 1, 0)
		self.weldLabel.TextXAlignment = Enum.TextXAlignment.Left
		self.weldLabel.Parent = self.weldContainer
		
		self.weldCheckbox = WidgetCheckbox()
		self.weldCheckbox:Parent(self.weldContainer)
		self.weldCheckbox.frame.Size = UDim2.new(0, 14, 0, 14)
		self.weldCheckbox.frame.Position = UDim2.new(1, -14, 0.5, -7)
		self.weldCheckbox:SetChecked(false)
		
		self.repeatBtn = self:MakeBtn('Repeat', function()
			if self.last and not self.repeatDebounce then
				self.repeatDebounce = true
				self:RepeatLast()
				self:UpdateUI()
				wait(0.1)
				self.repeatDebounce = false
			end
		end)
		
		self:UpdateUI()
	end
	
	function DupePartWindow:UpdateUI()
		if self:InProgress() then
			self.partSelection.Font = Enum.Font.GothamBold
			self.partSelection.Text = 'Click part to select'
		else
			self.partSelection.Font = Enum.Font.Gotham
			self.partSelection.Text = 'Click to select'
		end
		
		local repeatEnabled = self.last and not self.repeatDebounce
		
		self.repeatBtn.AutoButtonColor = repeatEnabled
		self.repeatBtn.TextTransparency = repeatEnabled and 0 or 0.25
		
		self:Height(self:ScrollHeight())
	end
	
	function DupePartWindow:Start()
		self.partSelect = startPartSelection(workspace.CurrentCamera, function(part, data)
			self:Cancel()
			self:UpdateUI()
			
			if part then
				--local multiplier = -self.numbersOnlyMultiplier:GetNumber()
				local weldToBuildZone = self.weldCheckbox.checked
				
				self:SetLast(part, --[[multiplier,]] weldToBuildZone)
				self:RepeatLast()
			end
		end, function(part)
			return part.Parent == MwsUtils:GetParts()
		end)
		
		self:UpdateUI()
	end
	
	function DupePartWindow:SetLast(part, --[[multiplier,]] weldToBuildZone)
		self.last = {part = part, --[[multiplier = multiplier,]] weldToBuildZone = weldToBuildZone}
	end
	
	function DupePartWindow:RepeatLast()
		local last = self.last
		local part = last.part
		--local multiplier = last.multiplier
		local weldToBuildZone = last.weldToBuildZone
		
		self.repeatDebounce = true
		self:UpdateUI()
		
		local part1, part2 = MwsUtils:CutPart(part, Vector3.new(0, 0, 0), Vector3.new(0, 0, 0))
		
		if part2 then
			if weldToBuildZone then
				local buildZone = MwsUtils:GetMyBuildZone()
				
				if buildZone then
					MwsUtils:Weld(part2, buildZone.Main, CFrame.new(0, -part2.Size.Y / 2, 0), CFrame.new(0, buildZone.Main.Size.Y / 2, 0), 0)
				end
			end
		end
		
		if part1 then
			self:SetLast(part1, --[[multiplier,]] weldToBuildZone)
		else
			self.last = nil
		end
		
		wait(0.25)
		self.repeatDebounce = false
		self:UpdateUI()
	end
	
	function DupePartWindow:Cancel()
		if self.partSelect then
			self.partSelect:Cancel()
			self.partSelect = nil
		end
	end
	
	function DupePartWindow:InProgress()
		return self.partSelect ~= nil
	end
	
	function DupePartWindow:Close()
		if self:InProgress() then
			self:Cancel()
		end
		
		self.__Super.Close(self)
	end
end

local dupePartWindow
local function func_ac()
local mouse = game:GetService('Players').LocalPlayer:GetMouse()
tool = Instance.new("Tool")
tool.RequiresHandle = false
tool.Name = "Abandon/Claim"
local function getzone()
local zone = mouse.Target
if zone.Name == 'Main' then return zone.Parent end
end
tool.Activated:connect(function()
local zone = getzone()
if zone.ZoneInfo and zone.ZoneInfo.OwnerPlayer.Value == nil then
zone.ZoneInfo.ClaimZone:InvokeServer()
else
zone.ZoneInfo.AbandonZone:InvokeServer()
end
end)
tool.Parent = game.Players.LocalPlayer.Backpack
end
local function func_dupePart()
	if dupePartWindow and not dupePartWindow:IsClosed() then return end
	dupePartWindow = DupePartWindow(screenGui)
end

local DetachPartWindow = Window:__Subclass('DetachPartWindow') do
	function DetachPartWindow:__SuperArgs(parent)
		return 'Detach Part', parent
	end
	
	function DetachPartWindow:__Construct()
		self.partSelection = makeGuiObject('TextButton')
		self.partSelection.Text = 'Click to select'
		self.partSelection.Size = UDim2.new(0, 150, 0, 20)
		self.partSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.partSelection.Position = UDim2.new(0.5, 0, 0, 8)
		self.partSelection.Parent = self.content
		
		self.partSelection.MouseButton1Click:Connect(function()
			if self:InProgress() then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self:Start()
			self:UpdateUI()
		end)
		
		self:UpdateUI()
	end
	
	function DetachPartWindow:UpdateUI()
		if self:InProgress() then
			self.partSelection.Font = Enum.Font.GothamBold
			self.partSelection.Text = 'Click part to select'
		else
			self.partSelection.Font = Enum.Font.Gotham
			self.partSelection.Text = 'Click to select'
		end
		
		self:Height(36)
	end
	
	function DetachPartWindow:Start()
		self.cancels = {}
		
		local selecting = startPartSelection(workspace.CurrentCamera, function(selected)
			self:Cancel()
			self:UpdateUI()
			
			for _, weld in ipairs(MwsUtils:GetPartWelds(selected)) do
				MwsUtils:RemoveWeld(weld)
				for i = 1, 3 do s.RunService.Stepped:Wait() end
			end
		end)
		
		self.cancels[1] = function()
			selecting:Cancel()
		end
	end
	
	function DetachPartWindow:Cancel()
		for _, cancel in ipairs(self.cancels) do
			cancel()
		end
		
		self.cancels = nil
	end
	
	function DetachPartWindow:InProgress()
		return self.cancels ~= nil
	end
	
	function DetachPartWindow:Close()
		if self:InProgress() then
			self:Cancel()
		end
		
		self.__Super.Close(self)
	end
end

local detachPartWindow

local function func_detachPart()
	if detachPartWindow and not detachPartWindow:IsClosed() then return end
	detachPartWindow = DetachPartWindow(screenGui)
end

local DecimateCreationWindow = Window:__Subclass('DecimateCreationWindow') do
	function DecimateCreationWindow:__SuperArgs(parent)
		return 'Decimate Creation', parent
	end
	
	function DecimateCreationWindow:__Construct()
		self.partSelection = makeGuiObject('TextButton')
		self.partSelection.Text = 'Click to select'
		self.partSelection.Size = UDim2.new(0, 150, 0, 20)
		self.partSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.partSelection.Position = UDim2.new(0.5, 0, 0, 8)
		self.partSelection.Parent = self.content
		
		self.partSelection.MouseButton1Click:Connect(function()
			if self.decimating then return end
			
			if self:InProgress() then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self:Start()
			self:UpdateUI()
		end)
		
		self.decimating = false
		
		self:UpdateUI()
	end
	
	function DecimateCreationWindow:UpdateUI()
		if self:InProgress() then
			self.partSelection.Font = Enum.Font.GothamBold
			self.partSelection.Text = 'Click part to select'
		else
			self.partSelection.Font = Enum.Font.Gotham
			self.partSelection.Text = self.decimating and 'Decimating...' or 'Click to select'
		end
		
		self.partSelection.AutoButtonColor = not self.decimating
		self.partSelection.TextTransparency = self.decimating and 0.25 or 0
		
		self:Height(36)
	end
	
	function DecimateCreationWindow:Start()
		self.cancels = {}
		
		local selecting = startPartSelection(workspace.CurrentCamera, function(selected)
			self:Cancel()
			
			if selected then
				self.decimating = true
				self:UpdateUI()
				
				for _, weld in ipairs(MwsUtils:GetCreationWelds(selected)) do
					MwsUtils:RemoveWeld(weld)
					for i = 1, 3 do s.RunService.Stepped:Wait() end
				end
				
				self.decimating = false
			end
			
			self:UpdateUI()	
		end)
		
		self.cancels[1] = function()
			selecting:Cancel()
		end
	end
	
	function DecimateCreationWindow:Cancel()
		for _, cancel in ipairs(self.cancels) do
			cancel()
		end
		
		self.cancels = nil
	end
	
	function DecimateCreationWindow:InProgress()
		return self.cancels ~= nil
	end
	
	function DecimateCreationWindow:Close()
		if self.decimating then return end
		
		if self:InProgress() then
			self:Cancel()
		end
		
		self.__Super.Close(self)
	end
end

local function func_swelds()
local WeldVisorTesting = Instance.new("Tool",game:GetService("Players").LocalPlayer.Backpack)
WeldVisorTesting.RequiresHandle = false
WeldVisorTesting.Name = "WeldVisor"
WeldVisorTesting.ToolTip = "-The BRUH TROLL Tool"
local Mouse = game:GetService("Players").LocalPlayer:GetMouse()
local EspTable = {}
local BillBoardsTable = {}
local TransparencyPartsTable = {}
local Toggle = false

WeldVisorTesting.Activated:Connect(function()

if Toggle == true then
Toggle = false
for i,m in ipairs(BillBoardsTable) do
m:Destroy()
end
BillBoardsTable = {}



for i,o in ipairs(EspTable) do
if o.ClassName == "Attachment" or o.ClassName == "Beam" or o.ClassName == "BoxHandleAdornment" then 
o:Destroy()
elseif o.ClassName == "Part" or o.ClassName == "MeshPart" then 
o.Transparency = 0
end
end
EspTable = {}
elseif Toggle == false then
Toggle = true
local Parts = Mouse.Target:GetConnectedParts(true)
for i,k in ipairs(Parts) do
if k:IsDescendantOf(game:GetService("Workspace").Welds) == true then -- k:IsDescendantOf(game:GetService("Workspace").Welds) == true and
table.insert(EspTable,k)
     local NewAdornee = Instance.new("BoxHandleAdornment")
     NewAdornee.Adornee = k
     NewAdornee.Size = k.Size + Vector3.new(0.1,0.1,0.1)
     NewAdornee.Transparency = 0
     NewAdornee.Parent = game:GetService("CoreGui")
     NewAdornee.ZIndex = 8
     NewAdornee.AlwaysOnTop = true
     NewAdornee.Color3 = Color3.new(255, 0, 0)
    local BillboardGui = Instance.new("BillboardGui",game:GetService("Players").LocalPlayer.PlayerGui)
    BillboardGui.AlwaysOnTop = true
    BillboardGui.Size = UDim2.new(0,15,0,15)
    BillboardGui.StudsOffset = Vector3.new(0,0,0)
    BillboardGui.Adornee = k
    BillboardGui.Active = true
    local Button = Instance.new("TextButton",BillboardGui)
    Button.Size = UDim2.new(0,15,0,15)
    Button.Text = ""
    Button.BackgroundColor3 = Color3.fromRGB(0,0,255)
    Button.BorderColor3 = Color3.fromRGB(0,0,255)
    Button.BackgroundTransparency = 0.5
    Button.ZIndex = 19
Button.MouseButton1Click:Connect(function()
for i,n in ipairs(k:GetChildren()) do
if n.ClassName == "Weld" then
n.Part0.Transparency = 0
n.Part1.Transparency = 0
end
end
NewAdornee:Destroy()
game:GetService("ReplicatedStorage").Remotes.ClientCut:FireServer(k,Vector3.new(0, 0, 0), Vector3.new(math.huge, math.huge, math.huge))
end)

table.insert(EspTable,NewAdornee)
table.insert(BillBoardsTable,BillboardGui)

for i,l in ipairs(k:GetChildren()) do
if l.ClassName == "Weld" then
local ATT0 = Instance.new("Attachment",l.Part0)
local ATT1 = Instance.new("Attachment",l.Part1)

               local Beam = Instance.new('Beam', l)
               Beam.Color = ColorSequence.new(Color3.fromRGB(255,0,0),Color3.fromRGB(255,0,0),Color3.fromRGB(255,0,0))
               Beam.FaceCamera = true
               Beam.Width0 = 0.1
               Beam.Width1 = 0.1
               Beam.Attachment0 = ATT0
               Beam.Attachment1 = ATT1
l.Part0.Transparency = 0.5
l.Part1.Transparency = 0.5
table.insert(TransparencyPartsTable,l.Part0)
table.insert(TransparencyPartsTable,l.Part1)
table.insert(EspTable,l.Part0)
table.insert(EspTable,l.Part1)
table.insert(EspTable,Beam)
table.insert(EspTable,ATT0)
table.insert(EspTable,ATT1)
end
end
end

end

end


end)
end

local decimateCreationWindow

local function func_decimateCreation()
	if decimateCreationWindow and not decimateCreationWindow:IsClosed() then return end
	decimateCreationWindow = DecimateCreationWindow(screenGui)
end

local WeldHingeWindow = Window:__Subclass('WeldHingeWindow') do
	function WeldHingeWindow:__SuperArgs(parent)
		return 'Weld Hinge', parent
	end
	
	function WeldHingeWindow:__Construct()
		self.itemSelection = makeGuiObject('TextButton')
		self.itemSelection.Text = 'Click to select'
		self.itemSelection.Size = UDim2.new(0, 150, 0, 20)
		self.itemSelection.AnchorPoint = Vector2.new(0.5, 0)
		self.itemSelection.Position = UDim2.new(0.5, 0, 0, 8)
		self.itemSelection.Parent = self.content
		
		self.itemSelection.MouseButton1Click:Connect(function()
			if self:InProgress() then
				self:Cancel()
				self:UpdateUI()
				
				return
			end
			
			self:Start()
			self:UpdateUI()
		end)
		
		self:UpdateUI()
	end
	
	function WeldHingeWindow:UpdateUI()
		if self:InProgress() then
			self.itemSelection.Font = Enum.Font.GothamBold
			self.itemSelection.Text = 'Click hinge to select'
		else
			self.itemSelection.Font = Enum.Font.Gotham
			self.itemSelection.Text = 'Click to select'
		end
		
		self:Height(36)
	end
	
	function WeldHingeWindow:Start()
		self.cancels = {}
		
		local selecting = startItemSelection(workspace.CurrentCamera, function(selected)
			self:Cancel()
			
			if selected then
				local hingePieces = MwsUtils:GetHingePieces(selected)
				local motorPieces = MwsUtils:GetMotorPieces(selected)
				
				local attachment0 = (hingePieces or motorPieces).attachment0
				local attachment1 = (hingePieces or motorPieces).attachment1
				
				local part0 = attachment0.Parent
				local part1 = attachment1.Parent
				
				local offset = CFrame.new(0, part0.Size.Y / 2, 0)
				local c0 = attachment0.CFrame * offset
				local c1 = attachment1.CFrame * offset
				
				MwsUtils:Weld(part0, part1, c0, c1, 0)
			end
			
			self:UpdateUI()
		end, function(item)
			return MwsUtils:IsHingeItem(item) or MwsUtils:IsMotorItem(item)
		end)
		
		self.cancels[1] = function()
			selecting:Cancel()
		end
	end
	
	function WeldHingeWindow:Cancel()
		for _, cancel in ipairs(self.cancels) do
			cancel()
		end
		
		self.cancels = nil
	end
	
	function WeldHingeWindow:InProgress()
		return self.cancels ~= nil
	end
	
	function WeldHingeWindow:Close()
		if self:InProgress() then
			self:Cancel()
		end
		
		self.__Super.Close(self)
	end
end

local function func_Humancar()
local s = {} do
	for _, v in pairs(game:GetChildren()) do
		if pcall(function() return v.ClassName end) and v.ClassName ~= '' and game:GetService(v.ClassName) then
			s[v.ClassName] = v
		end
	end
end
local makeWheel = function()
	local container = Instance.new('Model')
	container.Name = 'Wheel'
	
	local anchor = Instance.new('Part', container)
	anchor.Size = Vector3.new(1, 0.2, 1)
	anchor.Transparency = 1
	anchor.CanCollide = false
	anchor.Massless = true
	anchor.Name = 'Anchor'
	
	local hub = Instance.new('Part', container)
	hub.Size = Vector3.new(1, 1, 1)
	hub.Transparency = 1
	hub.CanCollide = false
	hub.Massless = true
	hub.Name = 'Hub'
	
	local wheel = Instance.new('Part', container)
	wheel.Shape = Enum.PartType.Cylinder
	wheel.Size = Vector3.new(1, 2, 2)
	wheel.CanCollide = false
	local regularPP = PhysicalProperties.new(wheel.Material)
	wheel.CustomPhysicalProperties = PhysicalProperties.new(regularPP.Density * 15, regularPP.Friction, regularPP.Elasticity)
	wheel.TopSurface = Enum.SurfaceType.Smooth
	wheel.BottomSurface = Enum.SurfaceType.Smooth
	wheel.BrickColor = BrickColor.new('Really black')
	wheel.Name = 'Wheel'
	
	local wheelCollisions = Instance.new('Part', wheel)
	wheelCollisions.Shape = Enum.PartType.Ball
	wheelCollisions.Size = Vector3.new(1, 1, 1) * math.min(wheel.Size.Y, wheel.Size.Z)
	wheelCollisions.Transparency = 1
	wheelCollisions.Massless = true
	local regularPP = PhysicalProperties.new(wheelCollisions.Material)
	wheelCollisions.CustomPhysicalProperties = PhysicalProperties.new(regularPP.Density, 2, 0)
	wheelCollisions.Name = 'Hitbox'
	local wheelCollisionsWeld = Instance.new('Weld', wheel)
	wheelCollisionsWeld.Part0 = wheelCollisions
	wheelCollisionsWeld.Part1 = wheel
	wheelCollisionsWeld.Name = 'Wheel'
	
	local hubcap = Instance.new('Part', wheel)
	hubcap.Shape = Enum.PartType.Cylinder
	hubcap.Size = wheel.Size + Vector3.new(0.01, -0.5, -0.5)
	hubcap.TopSurface = Enum.SurfaceType.Smooth
	hubcap.BottomSurface = Enum.SurfaceType.Smooth
	hubcap.BrickColor = BrickColor.new('Institutional white')
	hubcap.Massless = true
	hubcap.CanCollide = false
	hubcap.Name = 'Hubcap'
	local hubcapWeld = Instance.new('Weld', hubcap)
	hubcapWeld.Part0 = hubcap
	hubcapWeld.Part1 = wheel
	hubcapWeld.Name = 'Wheel'
	
	local anchorToHub = Instance.new('Attachment', anchor)
	anchorToHub.CFrame = CFrame.Angles(0, 0, -math.pi / 2)
	anchorToHub.Name = 'Hub'
	local hubToAnchor = Instance.new('Attachment', hub)
	hubToAnchor.CFrame = CFrame.Angles(0, 0, -math.pi / 2)
	hubToAnchor.Name = 'Anchor'
	local anchorCyl = Instance.new('CylindricalConstraint', anchor)
	anchorCyl.Attachment0 = anchorToHub
	anchorCyl.Attachment1 = hubToAnchor
	anchorCyl.LimitsEnabled = true
	anchorCyl.LowerLimit = 0.1
	anchorCyl.UpperLimit = 3
	anchorCyl.AngularLimitsEnabled = true
	anchorCyl.LowerAngle = 0
	anchorCyl.UpperAngle = 0
	anchorCyl.Name = 'Steer'
	local anchorSpring = Instance.new('SpringConstraint', anchor)
	anchorSpring.Attachment0 = anchorToHub
	anchorSpring.Attachment1 = hubToAnchor
	anchorSpring.FreeLength = 3
	anchorSpring.Stiffness = 5000
	anchorSpring.Damping = 200
	anchorSpring.Name = 'Suspension'
	
	local hubToWheel = Instance.new('Attachment', hub)
	hubToWheel.Name = 'Wheel'
	hubToWheel.CFrame = CFrame.new(0, 0.5, 0) * CFrame.Angles(0, math.pi, 0)
	local wheelToHub = Instance.new('Attachment', wheel)
	wheelToHub.Name = 'Hub'
	wheelToHub.CFrame = CFrame.Angles(0, math.pi, 0)
	local hubMotor = Instance.new('HingeConstraint', hub)
	hubMotor.Attachment0 = hubToWheel
	hubMotor.Attachment1 = wheelToHub
	hubMotor.ActuatorType = Enum.ActuatorType.Motor
	hubMotor.MotorMaxTorque = 0
	hubMotor.MotorMaxAcceleration = math.huge
	hubMotor.AngularSpeed = math.huge
	hubMotor.Name = 'Motor'
	
	container.PrimaryPart = anchor
	
	return {
		model = container,
		anchor = anchor,
		hub = hub,
		wheel = wheel,
		Steer = function(self, angle)
			anchorCyl.LowerAngle = angle
			anchorCyl.UpperAngle = angle
			
			return self
		end,
		SpringStiffness = function(self, stiffness)
			anchorSpring.Stiffness = stiffness
			
			return self
		end,
		SpringDamping = function(self, damping)
			anchorSpring.Damping = damping
			
			return self
		end,
		Torque = function(self, torque)
			hubMotor.MotorMaxTorque = torque
			
			return self
		end,
		Speed = function(self, speed)
			hubMotor.AngularVelocity = speed
			
			return self
		end,
		FixWheelPos = function(self)
			hub.CFrame = anchor.CFrame * CFrame.new(0, -2, 0)
			wheel.CFrame = hub.CFrame
		end,
		GetSpeed = function(self)
			return (wheel.RotVelocity - hub.RotVelocity):Dot(wheel.CFrame:VectorToWorldSpace(Vector3.new(-1, 0, 0)))
		end
	}
end

local makeCar = function()
	local container = Instance.new('Model')
	container.Name = 'Car'
	
	local base = Instance.new('Part', container)
	base.Transparency = 1
	base.CanCollide = false
	local regularPP = PhysicalProperties.new(base.Material)
	base.CustomPhysicalProperties = PhysicalProperties.new(regularPP.Density * 20, regularPP.Friction, regularPP.Elasticity)
	base.Name = 'Base'
	
	local wheels = {
		makeWheel(),
		makeWheel(),
		makeWheel(),
		makeWheel()
	}
	
	for fb = 1, 2 do
		for lr = 1, 2 do
			wheels[(fb - 1) * 2 + lr].model.Name = (fb == 1 and 'F' or 'B') .. (lr == 1 and 'L' or 'R') .. 'Wheel'
		end
	end
	
	local function recalculateSuspensionForces()
		local mass = base:GetMass()
		
		local weightDistribution = {}
		local stiffnesses = {}
		
		for i = 1, #wheels do
			weightDistribution[i] = mass / #wheels -- placeholder
			stiffnesses[i] = weightDistribution[i] * workspace.Gravity
		end
		
		for i, wheel in ipairs(wheels) do
			wheel:SpringStiffness(stiffnesses[i])
			wheel:SpringDamping(math.sqrt(stiffnesses[i] * weightDistribution[i]) * 2)
		end
	end
	
	base.Size = Vector3.new(4, 1, 6)
	
	local anchorWelds = {}
	
	for i, wheel in ipairs(wheels) do
		local weld = Instance.new('Weld', base)
		weld.Name = wheel.model.Name
		weld.Part0 = base
		weld.Part1 = wheel.anchor
		
		local fl = math.floor((i - 1) / 2)
		local lr = (i - 1) % 2
		
		weld.C0 = CFrame.new(-base.Size.X / 2 + base.Size.X * lr, 2, -base.Size.Z / 2 + base.Size.Z * fl)
		
		anchorWelds[i] = weld
		
		wheel.model.Parent = container
	end
	
	recalculateSuspensionForces()
	
	return {
		model = container,
		base = base,
		wheels = wheels,
		Steer = function(self, steer)
			for i = 1, 2 do
				wheels[i]:Steer(steer)
			end
			
			return self
		end,
		FixWheelPos = function(self)
			for _, wheel in ipairs(wheels) do
				wheel:FixWheelPos()
			end
			
			return self
		end
	}
end

local function create()
	local toReturn
	
	local player = s.Players.LocalPlayer
	local character = player.Character or player.CharacterAdded:Wait()
	while not character:FindFirstChildWhichIsA('Humanoid') do s.RunService.RenderStepped:Wait() end
	local humanoid = character:FindFirstChildWhichIsA('Humanoid')
	local rootPart = humanoid.RootPart
	
	local car = makeCar()
	car.model.Parent = character
	car.base.CFrame = rootPart.CFrame
	car:FixWheelPos()
	
	local welds = {}
	
	local function weld(root, part, offset)
		if not root or not part then return end
		
		local weld = Instance.new('Weld', root)
		weld.Part0 = part
		weld.Part1 = root
		weld.C1 = offset
		welds[#welds + 1] = weld
	end
	
	local deactivatedJoints = {}
	
	local function deactivateJoint(joint)
		if not joint then return end
		
		deactivatedJoints[joint] = joint.Part1
		joint.Part1 = nil
	end
	
	local pseudoWelds = {}
	
	local function pseudoWeld(root, part, joint, offset)
		if not root or not part then return end
		
		pseudoWelds[part] = {relativeTo = root, joint = joint, offset = offset}
	end
	
	pseudoWeld(car.base, rootPart, nil, CFrame.new(0, 1, 0) * CFrame.Angles(-math.pi / 2, 0, 0))
	
	humanoid.PlatformStand = true
	
	local forward = false
	local backward = false
	local left = false
	local right = false
	
	local steeringAngle = 30
	local brakeTorque = 30000
	local brakeSpeed = math.rad(15)
	local speed = 1000
	
	local function torqueCurve(speed)
		local abs = math.abs(speed)
		
		local launchTorque = 100000
		local launchCutoff = math.pi * 2 * 5 -- 5 revolution/second
		local baseTorque = 10000
		local torqueCutoff = math.pi * 2 * 50 -- 30 revolutions/second
		
		local currentTorque = math.max(baseTorque * (torqueCutoff - abs) / torqueCutoff, 0)
		local launchRatio = math.max((launchCutoff - abs) / launchCutoff, 0)
		local finalTorque = launchTorque * launchRatio + currentTorque * (1 - launchRatio)
		
		return finalTorque
	end
	
	local currentSteer = 0
	local steerTime = 1
	local returnTime = 0.1
	local lastSteer = tick()
	
	local function update()
		local forwardMult = (forward and 1 or 0) + (backward and -1 or 0)
		
		for _, wheel in ipairs(car.wheels) do
			wheel:Speed(speed * forwardMult)
			
			local currentSpeed = wheel:GetSpeed()
			
			if (forwardMult == 0 and math.abs(currentSpeed) < brakeSpeed) or (math.sign(forwardMult) ~= 0 and math.sign(currentSpeed) ~= math.sign(forwardMult) and math.abs(currentSpeed) > brakeSpeed) then
				wheel:Torque(brakeTorque)
			else
				wheel:Torque(torqueCurve(currentSpeed))
			end
		end
		
		local steerTarget = (right and 1 or 0) + (left and -1 or 0)
		local diff = (steerTarget - currentSteer)
		
		local t = tick()
		local unit = (t - lastSteer)
		lastSteer = t
		
		if math.abs(currentSteer + unit * math.sign(diff)) < math.abs(currentSteer) then
			unit = unit / returnTime
		else
			unit = unit / steerTime
		end
		
		if math.abs(diff) < unit then
			currentSteer = steerTarget
		else
			currentSteer = currentSteer + unit * math.sign(diff)
		end
		
		car:Steer(currentSteer * steeringAngle)
		
		for part, data in pairs(pseudoWelds) do
			if data.joint == nil then
				part.CFrame = data.relativeTo.CFrame * data.offset
				part.Velocity = data.relativeTo.Velocity
				part.RotVelocity = data.relativeTo.RotVelocity
			else
				data.joint.C1 = (data.joint.Part0.CFrame * data.joint.C0):ToObjectSpace(data.relativeTo.CFrame * data.offset):Inverse()
			end
		end
	end
	
	local function handleAction(action, state, input)
		local on = state == Enum.UserInputState.Begin
		
		if action == 'CarForward' then
			forward = on
		elseif action == 'CarBackward' then
			backward = on
		elseif action == 'CarLeft' then
			left = on
		elseif action == 'CarRight' then
			right = on
		elseif action == 'CarFlip' and on then
			car.base.CFrame = CFrame.new(0, 5, 0) * CFrame.new(car.base.CFrame.Position, car.base.CFrame.Position + car.base.CFrame.LookVector)
			car:FixWheelPos()
		elseif action == 'CarCancel' and on then
			toReturn:Destroy()
		end
		
		update()
		
		return Enum.ContextActionResult.Sink
	end
	
	local c1 = s.RunService.RenderStepped:Connect(update)
	local c2 = s.RunService.Stepped:Connect(update)
	local c3 = s.RunService.Heartbeat:Connect(update)
	
	s.ContextActionService:BindAction('CarForward', handleAction, false, Enum.PlayerActions.CharacterForward)
	s.ContextActionService:BindAction('CarBackward', handleAction, false, Enum.PlayerActions.CharacterBackward)
	s.ContextActionService:BindAction('CarLeft', handleAction, false, Enum.PlayerActions.CharacterLeft)
	s.ContextActionService:BindAction('CarRight', handleAction, false, Enum.PlayerActions.CharacterRight)
	s.ContextActionService:BindAction('CarFlip', handleAction, false, Enum.KeyCode.F) --Flip Car Key
	s.ContextActionService:BindAction('CarCancel', handleAction, false, Enum.KeyCode.V) --Deactivate Car Key
	
	toReturn = {
		Destroy = function()
			c1:Disconnect()
			c2:Disconnect()
			c3:Disconnect()
			s.ContextActionService:UnbindAction('CarForward')
			s.ContextActionService:UnbindAction('CarBackward')
			s.ContextActionService:UnbindAction('CarLeft')
			s.ContextActionService:UnbindAction('CarRight')
			s.ContextActionService:UnbindAction('CarFlip')
			s.ContextActionService:UnbindAction('CarCancel')
			car.model:Destroy()
			for _, weld in ipairs(welds) do weld:Destroy() end
			for joint, part1 in pairs(deactivatedJoints) do joint.Part1 = part1 end
			humanoid.PlatformStand = false
			rootPart.CFrame = CFrame.new(0, 8, 0) * CFrame.new(rootPart.CFrame.Position, rootPart.CFrame.Position + rootPart.CFrame.UpVector)
			
			s.ContextActionService:BindAction('CarReactivate', function(action, state, input)
				if state == Enum.UserInputState.Begin then
					s.ContextActionService:UnbindAction('CarReactivate')
					create()
				end
			end, false, Enum.KeyCode.V) --Activate Car Key
		end
	}
	
	humanoid.Died:Connect(toReturn.Destroy)
	
	return toReturn
end

s.ContextActionService:UnbindAction('CarReactivate')

create()
end
local function func_goodbye()
local mouse = game:GetService('Players').LocalPlayer:GetMouse()
tool = Instance.new("Tool")
tool.RequiresHandle = false
tool.Name = "Goodbye"
local function getpart()
local part = mouse.Target
if not part:IsDescendantOf(game.Workspace.Trees) and not part:IsDescendantOf(game.Workspace.Map) and not part:IsDescendantOf(game.Workspace.BuildZones) and not part:IsDescendantOf(game.Workspace.TrashCans) then return part end
end
tool.Activated:connect(function()
local part = getpart()
game.ReplicatedStorage.Remotes.ClientRequestOwnership:FireServer(part)
game.ReplicatedStorage.Remotes.ClientMakeWeldJoint:FireServer(workspace.BuildZones:GetChildren()[1]:FindFirstChildWhichIsA('BasePart'), part, CFrame.new(0, -1000, 0), CFrame.new(0, 0, 0), 0)
end)
tool.Parent = game.Players.LocalPlayer.Backpack
end
local function func_delete()
if _G.DestroyTeardown then _G.DestroyTeardown() end

local keydown = false
local key = '5'
local scale = 1

local scaleUp = ']'
local scaleDown = '['

local mouse = game:GetService('Players').LocalPlayer:GetMouse()
local clientCut = game:GetService('ReplicatedStorage'):WaitForChild('Remotes'):WaitForChild('ClientCut')
local renderStepped = game:GetService('RunService').RenderStepped

local conn1 = mouse.KeyDown:connect(function(pressed)
	if pressed == scaleUp then
		scale = scale + 1
		print(scale)
	elseif pressed == scaleDown then
		scale = scale - 1
		print(scale)
	elseif pressed == key then
		keydown = true

		local first = true

		while keydown do
			local target = mouse.Target

			if target then
				clientCut:FireServer(target, Vector3.new(0,0,0), Vector3.new(999,999,999), 0)
			end

			if first then
				wait(0.5)
				first = false
			end

			renderStepped:Wait()
		end
	end
end)

local conn2 = mouse.KeyUp:connect(function(pressed)
	if pressed == key then
		keydown = false
	end
end)

_G.DestroyTeardown=function()
	conn1:Disconnect()
	conn2:Disconnect()
	keydown = false
end
end
local function func_sk()
-- Instances:

local SK = Instance.new("ScreenGui")
local Frame = Instance.new("Frame")
local TextLabel = Instance.new("TextLabel")
local CloseBtn = Instance.new("TextButton")
local TextBox = Instance.new("TextBox")
local KillBtn = Instance.new("TextButton")

--Properties:

SK.Name = "SK"
SK.Parent = game.Players.LocalPlayer:WaitForChild("PlayerGui")
SK.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
SK.ResetOnSpawn = false

Frame.Parent = SK
Frame.AnchorPoint = Vector2.new(0.5, 0.5)
Frame.BackgroundColor3 = Color3.fromRGB(145, 145, 145)
Frame.BackgroundTransparency = 0.500
Frame.Position = UDim2.new(0.5, 0, 0.5, 0)
Frame.Size = UDim2.new(0, 200, 0, 70)

TextLabel.Parent = Frame
TextLabel.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
TextLabel.BackgroundTransparency = 0.500
TextLabel.Size = UDim2.new(0, 180, 0, 20)
TextLabel.Font = Enum.Font.SourceSansSemibold
TextLabel.Text = "SeatKiller GUI v1"
TextLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
TextLabel.TextScaled = true
TextLabel.TextSize = 14.000
TextLabel.TextWrapped = true

CloseBtn.Name = "CloseBtn"
CloseBtn.Parent = Frame
CloseBtn.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
CloseBtn.BackgroundTransparency = 0.200
CloseBtn.Position = UDim2.new(0.899999976, 0, 0, 0)
CloseBtn.Size = UDim2.new(0, 20, 0, 20)
CloseBtn.Font = Enum.Font.SourceSans
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
CloseBtn.TextScaled = true
CloseBtn.TextSize = 14.000
CloseBtn.TextWrapped = true

TextBox.Parent = Frame
TextBox.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextBox.BackgroundTransparency = 0.200
TextBox.Position = UDim2.new(0.0450000018, 0, 0.428571433, 0)
TextBox.Size = UDim2.new(0, 132, 0, 30)
TextBox.Font = Enum.Font.SourceSans
TextBox.Text = ""
TextBox.TextColor3 = Color3.fromRGB(0, 0, 0)
TextBox.TextSize = 14.000

KillBtn.Name = "KillBtn"
KillBtn.Parent = Frame
KillBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
KillBtn.BackgroundTransparency = 0.200
KillBtn.Position = UDim2.new(0.745000005, 0, 0.428571433, 0)
KillBtn.Size = UDim2.new(0, 43, 0, 30)
KillBtn.Font = Enum.Font.Jura
KillBtn.Text = "Kill"
KillBtn.TextColor3 = Color3.fromRGB(0, 0, 0)
KillBtn.TextScaled = true
KillBtn.TextSize = 14.000
KillBtn.TextWrapped = true

-- Scripts:

    local function findSeat()
      for _, lol in ipairs(workspace.Items:GetChildren()) do
        if lol.Name == 'Seat' and not lol:FindFirstChildWhichIsA('BasePart'):IsGrounded() and lol.Bindings:FindFirstChild('Binding') and lol.Bindings.Binding.KeyCode.Value == 279 then return lol end
    end
end

local function GGGGG_fake_script() -- drag
	local script = Instance.new('LocalScript', Frame)
local UserInputService = game:GetService("UserInputService")

local gui = script.Parent

local dragging
local dragInput
local dragStart
local startPos

local function update(input)
	local delta = input.Position - dragStart
	gui.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end

gui.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = input.Position
		startPos = gui.Position

		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then
				dragging = false
			end
		end)
	end
end)

gui.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
		dragInput = input
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if input == dragInput and dragging then
		update(input)
	end
end)

end
coroutine.wrap(GGGGG_fake_script)()

local function GFYUW_fake_script() -- CloseBtn.CloseScript 
	local script = Instance.new('LocalScript', CloseBtn)

	local Button = script.Parent
	Button.Activated:Connect(function()
		game.Players.LocalPlayer.PlayerGui.SK:Destroy()
	end)
end

coroutine.wrap(GFYUW_fake_script)()

local function FUCKU_fake_script() -- killbtn script
	local script = Instance.new('LocalScript', KillBtn)

	local Button = script.Parent
	Button.Activated:Connect(function()
local function victim()
for i, v in ipairs(game.Players:GetChildren()) do
if v.Name == TextBox.Text then return v end end
end
local char = victim().Character
local hum = char.Humanoid

if hum.Sit == true then
 game.ReplicatedStorage.Remotes.ClientRequestOwnership:FireServer(hum.SeatPart)
 game.ReplicatedStorage.Remotes.ClientMakeWeldJoint:FireServer(workspace.BuildZones:GetChildren()[1]:FindFirstChildWhichIsA('BasePart'), hum.SeatPart, CFrame.new(0, -1000, 0), CFrame.new(0, 0, 0), 0)
else
local A1 = "Load"
local A2 = "seatkiller"
local Event = game:GetService("ReplicatedStorage").Remotes.LoadSave
Event:InvokeServer(A1, A2)
wait(0.7)
while true do 
wait()
game.ReplicatedStorage.Remotes.ClientRequestOwnership:FireServer(findSeat().ControlSeat)
findSeat().ControlSeat.Position = char.HumanoidRootPart.Position+Vector3.new(0, -1, 0)
if hum.Sit == true then
game.ReplicatedStorage.Remotes.ClientMakeWeldJoint:FireServer(workspace.BuildZones:GetChildren()[1]:FindFirstChildWhichIsA('BasePart'), hum.SeatPart, CFrame.new(0, -1000, 0), CFrame.new(0, 0, 0), 0)
break
end
end
end
	end)
end
coroutine.wrap(FUCKU_fake_script)()
end
local function func_abandon()
   local buildZone = MwsUtils:GetMyBuildZone()
   local bac = buildZone.ZoneInfo.AbandonZone;
   bac:InvokeServer()
   end
local function func_fastdupe()
if _G.DestroyTeardown then _G.DestroyTeardown() end

local keydown = false
local key = '5'
local scale = 1

local scaleUp = ']'
local scaleDown = '['

local mouse = game:GetService('Players').LocalPlayer:GetMouse()
local clientCut = game:GetService('ReplicatedStorage'):WaitForChild('Remotes'):WaitForChild('ClientCut')
local renderStepped = game:GetService('RunService').RenderStepped

local conn1 = mouse.KeyDown:connect(function(pressed)
	if pressed == scaleUp then
		scale = scale + 1
		print(scale)
	elseif pressed == scaleDown then
		scale = scale - 1
		print(scale)
	elseif pressed == key then
		keydown = true

		local first = true

		while keydown do
			local target = mouse.Target

			if target then
				clientCut:FireServer(target, Vector3.new(0,0,0), Vector3.new(0,0,0), 0)
			end

			if first then
				wait(0.5)
				first = false
			end

			renderStepped:Wait()
		end
	end
end)

local conn2 = mouse.KeyUp:connect(function(pressed)
	if pressed == key then
		keydown = false
	end
end)

_G.DestroyTeardown=function()
	conn1:Disconnect()
	conn2:Disconnect()
	keydown = false
end
end
local function func_fling()
local SynapseXen_IlllIiIlI=select;local SynapseXen_ilIlliiIIllIlIi=string.byte;local SynapseXen_lIlIiil=string.sub;local SynapseXen_iiiiIilIIii=string.char;local SynapseXen_iIiilIiIllIlilII=type;local SynapseXen_lIllIiIllIIIIiii=table.concat;local unpack=unpack;local setmetatable=setmetatable;local pcall=pcall;local SynapseXen_lillIIili,SynapseXen_lliIiIilIii,SynapseXen_iiIIlliIIIiliil,SynapseXen_IiiilIIIIiIIIlI;if bit and bit.bxor then SynapseXen_lillIIili=bit.bxor;SynapseXen_lliIiIilIii=function(SynapseXen_lilIIlil,SynapseXen_lIlIlliliIiIiI)local SynapseXen_IIIliIlilIIIllIiI=SynapseXen_lillIIili(SynapseXen_lilIIlil,SynapseXen_lIlIlliliIiIiI)if SynapseXen_IIIliIlilIIIllIiI<0 then SynapseXen_IIIliIlilIIIllIiI=4294967296+SynapseXen_IIIliIlilIIIllIiI end;return SynapseXen_IIIliIlilIIIllIiI end else SynapseXen_lillIIili=function(SynapseXen_lilIIlil,SynapseXen_lIlIlliliIiIiI)local SynapseXen_iIiiIIiIli=function(SynapseXen_ililli,SynapseXen_lIIiI)return SynapseXen_ililli%(SynapseXen_lIIiI*2)>=SynapseXen_lIIiI end;local SynapseXen_lIIilliiliIIlI=0;for SynapseXen_IIillIiiiiIi=0,31 do SynapseXen_lIIilliiliIIlI=SynapseXen_lIIilliiliIIlI+(SynapseXen_iIiiIIiIli(SynapseXen_lilIIlil,2^SynapseXen_IIillIiiiiIi)~=SynapseXen_iIiiIIiIli(SynapseXen_lIlIlliliIiIiI,2^SynapseXen_IIillIiiiiIi)and 2^SynapseXen_IIillIiiiiIi or 0)end;return SynapseXen_lIIilliiliIIlI end;SynapseXen_lliIiIilIii=SynapseXen_lillIIili end;SynapseXen_iiIIlliIIIiliil=function(SynapseXen_IlIlllllIIIIlIilii,SynapseXen_llliIiiIii,SynapseXen_IIilIIIlliiiIliI)return(SynapseXen_IlIlllllIIIIlIilii+SynapseXen_llliIiiIii)%SynapseXen_IIilIIIlliiiIliI end;SynapseXen_IiiilIIIIiIIIlI=function(SynapseXen_IlIlllllIIIIlIilii,SynapseXen_llliIiiIii,SynapseXen_IIilIIIlliiiIliI)return(SynapseXen_IlIlllllIIIIlIilii-SynapseXen_llliIiiIii)%SynapseXen_IIilIIIlliiiIliI end;local function SynapseXen_iIiilli(SynapseXen_IIIliIlilIIIllIiI)if SynapseXen_IIIliIlilIIIllIiI<0 then SynapseXen_IIIliIlilIIIllIiI=4294967296+SynapseXen_IIIliIlilIIIllIiI end;return SynapseXen_IIIliIlilIIIllIiI end;local getfenv=getfenv;if not getfenv then getfenv=function()return _ENV end end;local SynapseXen_iIilIIIlllIIiIiiI={}local SynapseXen_IIliiIliIl={}local SynapseXen_IIiIllllliI;local SynapseXen_IliIIlIIlIlIlI;local SynapseXen_lllIiIIIil={}local SynapseXen_lliiililiIi={}for SynapseXen_IIillIiiiiIi=0,255 do local SynapseXen_lliIIlIli,SynapseXen_llllIIilIIIli=SynapseXen_iiiiIilIIii(SynapseXen_IIillIiiiiIi),SynapseXen_iiiiIilIIii(SynapseXen_IIillIiiiiIi,0)SynapseXen_lllIiIIIil[SynapseXen_lliIIlIli]=SynapseXen_llllIIilIIIli;SynapseXen_lliiililiIi[SynapseXen_llllIIilIIIli]=SynapseXen_lliIIlIli end;local function SynapseXen_iiIIlIiIIliIi(SynapseXen_llIlliliI,SynapseXen_iilIlliI,SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll)if SynapseXen_IIlIiIiiillIlI>=256 then SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll=0,SynapseXen_iIlllliIlIll+1;if SynapseXen_iIlllliIlIll>=256 then SynapseXen_iilIlliI={}SynapseXen_iIlllliIlIll=1 end end;SynapseXen_iilIlliI[SynapseXen_iiiiIilIIii(SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll)]=SynapseXen_llIlliliI;SynapseXen_IIlIiIiiillIlI=SynapseXen_IIlIiIiiillIlI+1;return SynapseXen_iilIlliI,SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll end;local function SynapseXen_ilIIl(SynapseXen_iliilIlliII)local function SynapseXen_iIiiliiiIIlIlll(SynapseXen_IIlIIlIi)local SynapseXen_iIlllliIlIll='ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'SynapseXen_IIlIIlIi=string.gsub(SynapseXen_IIlIIlIi,'[^'..SynapseXen_iIlllliIlIll..'=]','')return SynapseXen_IIlIIlIi:gsub('.',function(SynapseXen_IlIlllllIIIIlIilii)if SynapseXen_IlIlllllIIIIlIilii=='='then return''end;local SynapseXen_iiiIIIlIIIillllIlIi,SynapseXen_lIiIIlll='',SynapseXen_iIlllliIlIll:find(SynapseXen_IlIlllllIIIIlIilii)-1;for SynapseXen_IIillIiiiiIi=6,1,-1 do SynapseXen_iiiIIIlIIIillllIlIi=SynapseXen_iiiIIIlIIIillllIlIi..(SynapseXen_lIiIIlll%2^SynapseXen_IIillIiiiiIi-SynapseXen_lIiIIlll%2^(SynapseXen_IIillIiiiiIi-1)>0 and'1'or'0')end;return SynapseXen_iiiIIIlIIIillllIlIi end):gsub('%d%d%d?%d?%d?%d?%d?%d?',function(SynapseXen_IlIlllllIIIIlIilii)if#SynapseXen_IlIlllllIIIIlIilii~=8 then return''end;local SynapseXen_lIlIllll=0;for SynapseXen_IIillIiiiiIi=1,8 do SynapseXen_lIlIllll=SynapseXen_lIlIllll+(SynapseXen_IlIlllllIIIIlIilii:sub(SynapseXen_IIillIiiiiIi,SynapseXen_IIillIiiiiIi)=='1'and 2^(8-SynapseXen_IIillIiiiiIi)or 0)end;return string.char(SynapseXen_lIlIllll)end)end;SynapseXen_iliilIlliII=SynapseXen_iIiiliiiIIlIlll(SynapseXen_iliilIlliII)local SynapseXen_lilIiiiiiiiIlllI=SynapseXen_lIlIiil(SynapseXen_iliilIlliII,1,1)if SynapseXen_lilIiiiiiiiIlllI=="u"then return SynapseXen_lIlIiil(SynapseXen_iliilIlliII,2)elseif SynapseXen_lilIiiiiiiiIlllI~="c"then error("Synapse Xen - Failed to verify bytecode. Please make sure your Lua implementation supports non-null terminated strings.")end;SynapseXen_iliilIlliII=SynapseXen_lIlIiil(SynapseXen_iliilIlliII,2)local SynapseXen_llIllIllIiiIiiIilII=#SynapseXen_iliilIlliII;local SynapseXen_iilIlliI={}local SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll=0,1;local SynapseXen_lIIllllilliII={}local SynapseXen_IIIliIlilIIIllIiI=1;local SynapseXen_illIiliiII=SynapseXen_lIlIiil(SynapseXen_iliilIlliII,1,2)SynapseXen_lIIllllilliII[SynapseXen_IIIliIlilIIIllIiI]=SynapseXen_lliiililiIi[SynapseXen_illIiliiII]or SynapseXen_iilIlliI[SynapseXen_illIiliiII]SynapseXen_IIIliIlilIIIllIiI=SynapseXen_IIIliIlilIIIllIiI+1;for SynapseXen_IIillIiiiiIi=3,SynapseXen_llIllIllIiiIiiIilII,2 do local SynapseXen_liiilllIiIiIlii=SynapseXen_lIlIiil(SynapseXen_iliilIlliII,SynapseXen_IIillIiiiiIi,SynapseXen_IIillIiiiiIi+1)local SynapseXen_liiliII=SynapseXen_lliiililiIi[SynapseXen_illIiliiII]or SynapseXen_iilIlliI[SynapseXen_illIiliiII]if not SynapseXen_liiliII then error("Synapse Xen - Failed to verify bytecode. Please make sure your Lua implementation supports non-null terminated strings.")end;local SynapseXen_liIlIlIliIi=SynapseXen_lliiililiIi[SynapseXen_liiilllIiIiIlii]or SynapseXen_iilIlliI[SynapseXen_liiilllIiIiIlii]if SynapseXen_liIlIlIliIi then SynapseXen_lIIllllilliII[SynapseXen_IIIliIlilIIIllIiI]=SynapseXen_liIlIlIliIi;SynapseXen_IIIliIlilIIIllIiI=SynapseXen_IIIliIlilIIIllIiI+1;SynapseXen_iilIlliI,SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll=SynapseXen_iiIIlIiIIliIi(SynapseXen_liiliII..SynapseXen_lIlIiil(SynapseXen_liIlIlIliIi,1,1),SynapseXen_iilIlliI,SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll)else local SynapseXen_ilillIliIIllil=SynapseXen_liiliII..SynapseXen_lIlIiil(SynapseXen_liiliII,1,1)SynapseXen_lIIllllilliII[SynapseXen_IIIliIlilIIIllIiI]=SynapseXen_ilillIliIIllil;SynapseXen_IIIliIlilIIIllIiI=SynapseXen_IIIliIlilIIIllIiI+1;SynapseXen_iilIlliI,SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll=SynapseXen_iiIIlIiIIliIi(SynapseXen_ilillIliIIllil,SynapseXen_iilIlliI,SynapseXen_IIlIiIiiillIlI,SynapseXen_iIlllliIlIll)end;SynapseXen_illIiliiII=SynapseXen_liiilllIiIiIlii end;return SynapseXen_lIllIiIllIIIIiii(SynapseXen_lIIllllilliII)end;local function SynapseXen_IIIlliIllli(SynapseXen_IIlIl,SynapseXen_IIIIllllll,SynapseXen_lIIIIIlliIIIliIIlI)if SynapseXen_lIIIIIlliIIIliIIlI then local SynapseXen_lillIIill=SynapseXen_IIlIl/2^(SynapseXen_IIIIllllll-1)%2^(SynapseXen_lIIIIIlliIIIliIIlI-1-(SynapseXen_IIIIllllll-1)+1)return SynapseXen_lillIIill-SynapseXen_lillIIill%1 else local SynapseXen_lilllIiIiill=2^(SynapseXen_IIIIllllll-1)if SynapseXen_IIlIl%(SynapseXen_lilllIiIiill+SynapseXen_lilllIiIiill)>=SynapseXen_lilllIiIiill then return 1 else return 0 end end end;local function SynapseXen_IiiIlilIililI()local SynapseXen_ilIilIlIlilii=SynapseXen_lillIIili(284364308,SynapseXen_IliIIlIIlIlIlI)while true do if SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(284404033,SynapseXen_IliIIlIIlIlIlI)then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi+34874,SynapseXen_IiiiiiIlIliilillil-41753)-SynapseXen_lillIIili(3981732499,SynapseXen_IIliiIliIl[3])end;SynapseXen_ilIilIlIlilii=SynapseXen_ilIilIlIlilii-SynapseXen_lillIIili(2847761747,SynapseXen_IliIIlIIlIlIlI)elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(284405531,SynapseXen_IliIIlIIlIlIlI)then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi+22024,SynapseXen_IiiiiiIlIliilillil+26736)-SynapseXen_lillIIili(2847751205,SynapseXen_IliIIlIIlIlIlI)end;SynapseXen_ilIilIlIlilii=SynapseXen_lillIIili(SynapseXen_ilIilIlIlilii,SynapseXen_lillIIili(2518682266,SynapseXen_IliIIlIIlIlIlI))elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(2395142449,SynapseXen_IIliiIliIl[12])then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi+49477,SynapseXen_IiiiiiIlIliilillil+19239)-SynapseXen_lillIIili(2847736413,SynapseXen_IliIIlIIlIlIlI)end;SynapseXen_ilIilIlIlilii=SynapseXen_ilIilIlIlilii+SynapseXen_lillIIili(135359769,SynapseXen_IIliiIliIl[12])elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(795419322,SynapseXen_IliIIlIIlIlIlI)then return elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(4093999313,SynapseXen_IIliiIliIl[9])then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi-5405,SynapseXen_IiiiiiIlIliilillil-45347)+SynapseXen_lillIIili(680601543,SynapseXen_IIliiIliIl[5])end;SynapseXen_ilIilIlIlilii=SynapseXen_ilIilIlIlilii+SynapseXen_lillIIili(2352500206,SynapseXen_IIliiIliIl[13])elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(284408934,SynapseXen_IliIIlIIlIlIlI)then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi-27275,SynapseXen_IiiiiiIlIliilillil+10717)+SynapseXen_lillIIili(256676943,SynapseXen_IIliiIliIl[11])end;SynapseXen_ilIilIlIlilii=SynapseXen_ilIilIlIlilii+SynapseXen_lillIIili(2847767341,SynapseXen_IliIIlIIlIlIlI)elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(795791881,SynapseXen_IliIIlIIlIlIlI)then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi-3118,SynapseXen_IiiiiiIlIliilillil-33791)+SynapseXen_lillIIili(1296815834,SynapseXen_IIliiIliIl[9])end;SynapseXen_ilIilIlIlilii=SynapseXen_ilIilIlIlilii+SynapseXen_lillIIili(848689883,SynapseXen_IIliiIliIl[1])elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(284364308,SynapseXen_IliIIlIIlIlIlI)then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi-26550,SynapseXen_IiiiiiIlIliilillil+30778)-SynapseXen_lillIIili(2847739002,SynapseXen_IliIIlIIlIlIlI)end;SynapseXen_ilIilIlIlilii=SynapseXen_ilIilIlIlilii+SynapseXen_lillIIili(2847759667,SynapseXen_IliIIlIIlIlIlI)elseif SynapseXen_ilIilIlIlilii==SynapseXen_lillIIili(426191173,SynapseXen_IIliiIliIl[7])then SynapseXen_IIiIllllliI=function(SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil)return SynapseXen_lillIIili(SynapseXen_IliIIilIlIIIilliiIi+5824,SynapseXen_IiiiiiIlIliilillil+47369)+SynapseXen_lillIIili(2687086905,SynapseXen_IIliiIliIl[7])end;SynapseXen_ilIilIlIlilii=SynapseXen_ilIilIlIlilii+SynapseXen_lillIIili(2847768473,SynapseXen_IliIIlIIlIlIlI)end end end;local function SynapseXen_IlliI(SynapseXen_illIlliIliili)local SynapseXen_IIiiillIiIiIil=1;local SynapseXen_iiIilllliiiIiIlIli;local SynapseXen_IliillI;local function SynapseXen_lIllIliIliIIiiii()local SynapseXen_iilIl=SynapseXen_ilIlliiIIllIlIi(SynapseXen_illIlliIliili,SynapseXen_IIiiillIiIiIil,SynapseXen_IIiiillIiIiIil)SynapseXen_IIiiillIiIiIil=SynapseXen_IIiiillIiIiIil+1;return SynapseXen_iilIl end;local function SynapseXen_IliiIIlIlliiilliiii()local SynapseXen_iiIlililIi,SynapseXen_IliIIilIlIIIilliiIi,SynapseXen_IiiiiiIlIliilillil,SynapseXen_lIillIlilIl=SynapseXen_ilIlliiIIllIlIi(SynapseXen_illIlliIliili,SynapseXen_IIiiillIiIiIil,SynapseXen_IIiiillIiIiIil+3)SynapseXen_IIiiillIiIiIil=SynapseXen_IIiiillIiIiIil+4;return SynapseXen_lIillIlilIl*16777216+SynapseXen_IiiiiiIlIliilillil*65536+SynapseXen_IliIIilIlIIIilliiIi*256+SynapseXen_iiIlililIi end;local function SynapseXen_iiiIIIlllli()return SynapseXen_IliiIIlIlliiilliiii()*4294967296+SynapseXen_IliiIIlIlliiilliiii()end;local function SynapseXen_IIiliIi()local SynapseXen_IlllIiilillliIli=SynapseXen_lliIiIilIii(SynapseXen_IliiIIlIlliiilliiii(),SynapseXen_iIilIIIlllIIiIiiI[1440126736]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="pain is gonna use the backspace method on xen"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1343219140,2349585778)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3278919165,3278990276)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1440126736]=SynapseXen_lillIIili(SynapseXen_lillIIili(2543734118,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1137735963,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2334337529,2884819927,293762280,1321717865,817141670,2602898351,1985085227,2970970230}return SynapseXen_iIilIIIlllIIiIiiI[1440126736]end)(3889))local SynapseXen_IllIiIlIiliIIIlIIII=SynapseXen_lliIiIilIii(SynapseXen_IliiIIlIlliiilliiii(),SynapseXen_iIilIIIlllIIiIiiI[3211022925]or(function()local SynapseXen_IlIlllllIIIIlIilii="level 1 crook = luraph, level 100 boss = xen"SynapseXen_iIilIIIlllIIiIiiI[3211022925]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3303077598,2576428723),SynapseXen_lillIIili(1280567973,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2826127048,2641967372,502920086,4196071680}return SynapseXen_iIilIIIlllIIiIiiI[3211022925]end)())local SynapseXen_liIIli=1;local SynapseXen_llIilII=SynapseXen_IIIlliIllli(SynapseXen_IllIiIlIiliIIIlIIII,1,20)*2^32+SynapseXen_IlllIiilillliIli;local SynapseXen_IliiiIlIiili=SynapseXen_IIIlliIllli(SynapseXen_IllIiIlIiliIIIlIIII,21,31)local SynapseXen_liIilillIiIIIIill=(-1)^SynapseXen_IIIlliIllli(SynapseXen_IllIiIlIiliIIIlIIII,32)if SynapseXen_IliiiIlIiili==0 then if SynapseXen_llIilII==0 then return SynapseXen_liIilillIiIIIIill*0 else SynapseXen_IliiiIlIiili=1;SynapseXen_liIIli=0 end elseif SynapseXen_IliiiIlIiili==2047 then if SynapseXen_llIilII==0 then return SynapseXen_liIilillIiIIIIill*1/0 else return SynapseXen_liIilillIiIIIIill*0/0 end end;return math.ldexp(SynapseXen_liIilillIiIIIIill,SynapseXen_IliiiIlIiili-1023)*(SynapseXen_liIIli+SynapseXen_llIilII/2^52)end;local function SynapseXen_lliiilliilIillIl(SynapseXen_ilIIi)local SynapseXen_llilI;if SynapseXen_ilIIi then SynapseXen_llilI=SynapseXen_lIlIiil(SynapseXen_illIlliIliili,SynapseXen_IIiiillIiIiIil,SynapseXen_IIiiillIiIiIil+SynapseXen_ilIIi-1)SynapseXen_IIiiillIiIiIil=SynapseXen_IIiiillIiIiIil+SynapseXen_ilIIi else SynapseXen_ilIIi=SynapseXen_iiIilllliiiIiIlIli()if SynapseXen_ilIIi==0 then return""end;SynapseXen_llilI=SynapseXen_lIlIiil(SynapseXen_illIlliIliili,SynapseXen_IIiiillIiIiIil,SynapseXen_IIiiillIiIiIil+SynapseXen_ilIIi-1)SynapseXen_IIiiillIiIiIil=SynapseXen_IIiiillIiIiIil+SynapseXen_ilIIi end;return SynapseXen_llilI end;local function SynapseXen_IlIIiIIlIliilI(SynapseXen_llilI)local SynapseXen_lillIIill={}for SynapseXen_IIillIiiiiIi=1,#SynapseXen_llilI do local SynapseXen_lllil=SynapseXen_llilI:sub(SynapseXen_IIillIiiiiIi,SynapseXen_IIillIiiiiIi)SynapseXen_lillIIill[#SynapseXen_lillIIill+1]=string.char(SynapseXen_lillIIili(string.byte(SynapseXen_lllil),SynapseXen_iIilIIIlllIIiIiiI[721508056]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi xen crashes on my axon paste plz help"SynapseXen_iIilIIIlllIIiIiiI[721508056]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(824479739,3911864817),SynapseXen_lillIIili(1119737800,SynapseXen_IIliiIliIl[8]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2702924692,2265305070,2925851808,1242549131,3390075396}return SynapseXen_iIilIIIlllIIiIiiI[721508056]end)()))end;return table.concat(SynapseXen_lillIIill)end;local function SynapseXen_IlllliiIlIiiIiillli()local SynapseXen_iIllilIiiIiIiIIIii={}local SynapseXen_IIilIIiiIIIiiliIil={}local SynapseXen_IiiIiii={}local SynapseXen_lIiIlIIilIIilililll={[SynapseXen_iIilIIIlllIIiIiiI[2212407899]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="print(bytecode)"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2396639053,3595037753)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3308081993,3308155260)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2212407899]=SynapseXen_lillIIili(SynapseXen_lillIIili(2339313654,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(597171454,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1394972345,2998539617,1137961080,4057727064}return SynapseXen_iIilIIIlllIIiIiiI[2212407899]end)({},"ilIllliiIl","il",{},"iIliilIilliilIIll","lIiillIiIiIillI","lilllliIIIIiIIl",12564)]=SynapseXen_iIllilIiiIiIiIIIii,[SynapseXen_iIilIIIlllIIiIiiI[1922386785]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen best rerubi paste"SynapseXen_iIilIIIlllIIiIiiI[1922386785]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(473211507,4251830984),SynapseXen_lillIIili(1714242037,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1361715766,325687812,3200677552}return SynapseXen_iIilIIIlllIIiIiiI[1922386785]end)()]=SynapseXen_IIilIIiiIIIiiliIil,[SynapseXen_iIilIIIlllIIiIiiI[2162822524]or(function()local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"SynapseXen_iIilIIIlllIIiIiiI[2162822524]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3356861403,1494227722),SynapseXen_lillIIili(1110672938,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1430253386,3056899946,3905974235,366285879}return SynapseXen_iIilIIIlllIIiIiiI[2162822524]end)()]=SynapseXen_IiiIiii}SynapseXen_IliiIIlIlliiilliiii()for SynapseXen_llllIiIIlliIilIlIl=1,SynapseXen_lillIIili(SynapseXen_IliillI(),SynapseXen_iIilIIIlllIIiIiiI[282356216]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="xen detects custom getfenv"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2090085135,4212669103)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1033743266,1033806255)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[282356216]=SynapseXen_lillIIili(SynapseXen_lillIIili(641516807,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1888434605,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{41977087}return SynapseXen_iIilIIIlllIIiIiiI[282356216]end)({},{},8649,{},"IiIiIiiliIl","illIl"))do local SynapseXen_iIiilIiIllIlilII=SynapseXen_lIllIliIliIIiiii()local SynapseXen_iIlliiIliilIllIlIiii;if SynapseXen_iIiilIiIllIlilII==(SynapseXen_iIilIIIlllIIiIiiI[2052662236]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen doesn't come with instance caching, sorry superskater"SynapseXen_iIilIIIlllIIiIiiI[2052662236]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1675364265,1271116335),SynapseXen_lillIIili(2174994813,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{591639320,2644320752,3891782733,2957588907,3604658163,394627368}return SynapseXen_iIilIIIlllIIiIiiI[2052662236]end)())then SynapseXen_iIlliiIliilIllIlIiii=SynapseXen_lIllIliIliIIiiii()~=0 elseif SynapseXen_iIiilIiIllIlilII==(SynapseXen_iIilIIIlllIIiIiiI[2524334387]or(function()local SynapseXen_IlIlllllIIIIlIilii="so if you'we nyot awawe of expwoiting by this point, you've pwobabwy been wiving undew a wock that the pionyeews used to wide fow miwes. wobwox is often seen as an expwoit-infested gwound by most fwom the suwface, awthough this isn't the case."SynapseXen_iIilIIIlllIIiIiiI[2524334387]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1847833682,2492438872),SynapseXen_lillIIili(3811137750,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2063882560,3095316416,2189512107,2648769554,3243308939}return SynapseXen_iIilIIIlllIIiIiiI[2524334387]end)())then SynapseXen_iIlliiIliilIllIlIiii=SynapseXen_IIiliIi()elseif SynapseXen_iIiilIiIllIlilII==(SynapseXen_iIilIIIlllIIiIiiI[4135277827]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="pain exist is gonna connect the dots of xen"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(846500174,4054256408)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1822998072,1823060421)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4135277827]=SynapseXen_lillIIili(SynapseXen_lillIIili(504842215,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1954218057,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2298031181,4126522810,4283943206,3535639327,3724955446}return SynapseXen_iIilIIIlllIIiIiiI[4135277827]end)(4325,{},{},8352,"ililIliIliIliIlllii","IliiIlI",4528))then SynapseXen_iIlliiIliilIllIlIiii=SynapseXen_lIlIiil(SynapseXen_IlIIiIIlIliilI(SynapseXen_lliiilliilIillIl()),1,-2)end;SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_llllIiIIlliIilIlIl-1]=SynapseXen_iIlliiIliilIllIlIiii end;SynapseXen_IliiIIlIlliiilliiii()SynapseXen_lIiIlIIilIIilililll[982698750]=SynapseXen_lillIIili(SynapseXen_lIllIliIliIIiiii(),SynapseXen_iIilIIIlllIIiIiiI[2561653520]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="imagine using some lua minifier tool and thinking you are a badass"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(291489342,3618134116)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(4038984417,4038992222)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2561653520]=SynapseXen_lillIIili(SynapseXen_lillIIili(742933861,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1127390714,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4234956687,3371416095,1740518164,3321787374,2268843585,541777922,1407030107}return SynapseXen_iIilIIIlllIIiIiiI[2561653520]end)("lIlIIIilIllIiil","IIIilIiIllIl"))SynapseXen_lIiIlIIilIIilililll[1481819974]=SynapseXen_lillIIili(SynapseXen_lIllIliIliIIiiii(),SynapseXen_iIilIIIlllIIiIiiI[240322654]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="https://twitter.com/Ripull_RBLX/status/1059334518581145603"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3979630598,3117893529)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1603159501,1603228816)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[240322654]=SynapseXen_lillIIili(SynapseXen_lillIIili(448370884,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3890619272,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1129592574,543698809,2898909455,2347671769,3429198049}return SynapseXen_iIilIIIlllIIiIiiI[240322654]end)({},{},{},{},"iIIIiiii",{},2111))SynapseXen_lIllIliIliIIiiii()SynapseXen_lIllIliIliIIiiii()for SynapseXen_llllIiIIlliIilIlIl=1,SynapseXen_lillIIili(SynapseXen_IliillI(),SynapseXen_iIilIIIlllIIiIiiI[3412562986]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi devforum"SynapseXen_iIilIIIlllIIiIiiI[3412562986]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3843295250,28874054),SynapseXen_lillIIili(1069920867,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2077755263}return SynapseXen_iIilIIIlllIIiIiiI[3412562986]end)())do SynapseXen_IliiIIlIlliiilliiii()local SynapseXen_iiliiiliiiiIiIIilI=SynapseXen_lillIIili(SynapseXen_IliiIIlIlliiilliiii(),SynapseXen_iIilIIIlllIIiIiiI[667672391]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="inb4 posted on exploit reports section"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2911019330,669009576)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2160258499,2160298656)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[667672391]=SynapseXen_lillIIili(SynapseXen_lillIIili(2272293244,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1575376223,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1623494557}return SynapseXen_iIilIIIlllIIiIiiI[667672391]end)("iliiiliilll",{},{},{},11784))local SynapseXen_IIiilliIIliiiliI=SynapseXen_lIllIliIliIIiiii()local SynapseXen_iIiilIiIllIlilII=SynapseXen_lIllIliIliIIiiii()SynapseXen_IliiIIlIlliiilliiii()local SynapseXen_iIIiillIiIIililIiiI={[638088808]=SynapseXen_iiliiiliiiiIiIIilI,[1902504146]=SynapseXen_IIiilliIIliiiliI,[1968668134]=SynapseXen_IIIlliIllli(SynapseXen_iiliiiliiiiIiIIilI,1,6),[1997878473]=SynapseXen_IIIlliIllli(SynapseXen_iiliiiliiiiIiIIilI,7,14)}if SynapseXen_iIiilIiIllIlilII==(SynapseXen_iIilIIIlllIIiIiiI[1998549184]or(function()local SynapseXen_IlIlllllIIIIlIilii="sponsored by ironbrew, jk xen is better"SynapseXen_iIilIIIlllIIiIiiI[1998549184]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3521300775,2085958405),SynapseXen_lillIIili(877588070,SynapseXen_IIliiIliIl[6]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{238252679,3806879448}return SynapseXen_iIilIIIlllIIiIiiI[1998549184]end)())then SynapseXen_iIIiillIiIIililIiiI[31674126]=SynapseXen_IIIlliIllli(SynapseXen_iiliiiliiiiIiIIilI,24,32)SynapseXen_iIIiillIiIIililIiiI[233588846]=SynapseXen_IIIlliIllli(SynapseXen_iiliiiliiiiIiIIilI,15,23)elseif SynapseXen_iIiilIiIllIlilII==(SynapseXen_iIilIIIlllIIiIiiI[2543233633]or(function()local SynapseXen_IlIlllllIIIIlIilii="sometimes it be like that"SynapseXen_iIilIIIlllIIiIiiI[2543233633]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2570731749,3589049487),SynapseXen_lillIIili(3848939723,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3901229943,1132148614,506048410,2945221225,3053043914}return SynapseXen_iIilIIIlllIIiIiiI[2543233633]end)())then SynapseXen_iIIiillIiIIililIiiI[518845294]=SynapseXen_IIIlliIllli(SynapseXen_iiliiiliiiiIiIIilI,15,32)elseif SynapseXen_iIiilIiIllIlilII==(SynapseXen_iIilIIIlllIIiIiiI[619595679]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="wait for someone on devforum to say they are gonna deobfuscate this"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1056243955,1263028247)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3792209668,3792272745)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[619595679]=SynapseXen_lillIIili(SynapseXen_lillIIili(3065241080,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1790065835,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1960274165,3034191123,2970112628,1877519216,793471082,2137585967,326989819,85689030}return SynapseXen_iIilIIIlllIIiIiiI[619595679]end)(14826,{},"iIIlIililIlliIilI"))then SynapseXen_iIIiillIiIIililIiiI[1050029882]=SynapseXen_IIIlliIllli(SynapseXen_iiliiiliiiiIiIIilI,15,32)-131071 end;SynapseXen_IiiIiii[SynapseXen_llllIiIIlliIilIlIl]=SynapseXen_iIIiillIiIIililIiiI end;SynapseXen_IliiIIlIlliiilliiii()for SynapseXen_llllIiIIlliIilIlIl=1,SynapseXen_lillIIili(SynapseXen_IliillI(),SynapseXen_iIilIIIlllIIiIiiI[168783539]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi my 2.5mb script doesn't work with xen please help"SynapseXen_iIilIIIlllIIiIiiI[168783539]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2198007548,1459610271),SynapseXen_lillIIili(3461167891,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3635145773,444313887,709418005,1493337136,4079348143,2979012919,3408049469,1887477170}return SynapseXen_iIilIIIlllIIiIiiI[168783539]end)())do SynapseXen_iIllilIiiIiIiIIIii[SynapseXen_llllIiIIlliIilIlIl-1]=SynapseXen_IlllliiIlIiiIiillli()end;return SynapseXen_lIiIlIIilIIilililll end;do assert(SynapseXen_lliiilliilIillIl(4)=="\27Xen","Synapse Xen - Failed to verify bytecode. Please make sure your Lua implementation supports non-null terminated strings.")SynapseXen_IliillI=SynapseXen_IliiIIlIlliiilliiii;SynapseXen_iiIilllliiiIiIlIli=SynapseXen_IliiIIlIlliiilliiii;local SynapseXen_iIIlilIl=SynapseXen_lliiilliilIillIl()SynapseXen_lIllIliIliIIiiii()SynapseXen_IliIIlIIlIlIlI=SynapseXen_iIiilli(SynapseXen_IliillI())SynapseXen_lIllIliIliIIiiii()SynapseXen_lIllIliIliIIiiii()local SynapseXen_iiliiIillIlliiIiIII=0;for SynapseXen_IIillIiiiiIi=1,#SynapseXen_iIIlilIl do local SynapseXen_lllil=SynapseXen_iIIlilIl:sub(SynapseXen_IIillIiiiiIi,SynapseXen_IIillIiiiiIi)SynapseXen_iiliiIillIlliiIiIII=SynapseXen_iiliiIillIlliiIiIII+string.byte(SynapseXen_lllil)end;SynapseXen_iiliiIillIlliiIiIII=SynapseXen_lillIIili(SynapseXen_iiliiIillIlliiIiIII,SynapseXen_IliIIlIIlIlIlI)for SynapseXen_llllIiIIlliIilIlIl=1,SynapseXen_lIllIliIliIIiiii()do SynapseXen_IIliiIliIl[SynapseXen_llllIiIIlliIilIlIl]=SynapseXen_lliIiIilIii(SynapseXen_IliillI(),SynapseXen_iiliiIillIlliiIiIII)end;SynapseXen_IiiIlilIililI()end;return SynapseXen_IlllliiIlIiiIiillli()end;local function SynapseXen_IillIiIlIlIlIlill(...)return SynapseXen_IlllIiIlI('#',...),{...}end;local function SynapseXen_iliiiIllIiliiiI(SynapseXen_lIiIlIIilIIilililll,SynapseXen_iiillIIIlIi,SynapseXen_lIlIiiIiIi)local SynapseXen_IIilIIiiIIIiiliIil=SynapseXen_lIiIlIIilIIilililll[SynapseXen_iIilIIIlllIIiIiiI[1922386785]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen best rerubi paste"SynapseXen_iIilIIIlllIIiIiiI[1922386785]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(473211507,4251830984),SynapseXen_lillIIili(1714242037,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1361715766,325687812,3200677552}return SynapseXen_iIilIIIlllIIiIiiI[1922386785]end)()]local SynapseXen_iIllilIiiIiIiIIIii=SynapseXen_lIiIlIIilIIilililll[SynapseXen_iIilIIIlllIIiIiiI[2212407899]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="print(bytecode)"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2396639053,3595037753)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3308081993,3308155260)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2212407899]=SynapseXen_lillIIili(SynapseXen_lillIIili(2339313654,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(597171454,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1394972345,2998539617,1137961080,4057727064}return SynapseXen_iIilIIIlllIIiIiiI[2212407899]end)({},"ilIllliiIl","il",{},"iIliilIilliilIIll","lIiillIiIiIillI","lilllliIIIIiIIl",12564)]local SynapseXen_IiiIiii=SynapseXen_lIiIlIIilIIilililll[SynapseXen_iIilIIIlllIIiIiiI[2162822524]or(function()local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"SynapseXen_iIilIIIlllIIiIiiI[2162822524]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3356861403,1494227722),SynapseXen_lillIIili(1110672938,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1430253386,3056899946,3905974235,366285879}return SynapseXen_iIilIIIlllIIiIiiI[2162822524]end)()]return function(...)local SynapseXen_iilliIiiiliillIIIllI,SynapseXen_lllilIiIill=1,-1;local SynapseXen_liilIiiI,SynapseXen_IlliiilIIilIli={},SynapseXen_IlllIiIlI('#',...)-1;local SynapseXen_ilIIiIIIllIli=0;local SynapseXen_IlllliiIlillIl={}local SynapseXen_IilIIliiiliII={}local SynapseXen_IliliIlilllliIIlll=setmetatable({},{__index=SynapseXen_IlllliiIlillIl,__newindex=function(SynapseXen_lllllIIIiI,SynapseXen_iIIlilllIIi,SynapseXen_lIIilliiiIIl)if SynapseXen_iIIlilllIIi>SynapseXen_lllilIiIill then SynapseXen_lllilIiIill=SynapseXen_iIIlilllIIi end;SynapseXen_IlllliiIlillIl[SynapseXen_iIIlilllIIi]=SynapseXen_lIIilliiiIIl end})local function SynapseXen_iilIIllIlIiIlilIill()local SynapseXen_iIIiillIiIIililIiiI,SynapseXen_IIllilI;while true do SynapseXen_iIIiillIiIIililIiiI=SynapseXen_IiiIiii[SynapseXen_iilliIiiiliillIIIllI]SynapseXen_IIllilI=SynapseXen_iIIiillIiIIililIiiI[1902504146]SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1;if SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3323382624]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="luraph better then xen bros :pensive:"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1357483731,719659571)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3840401461,3840398871)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3323382624]=SynapseXen_lillIIili(SynapseXen_lillIIili(1151349454,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2534646601,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2543926762,633194745,877280197,347272298,128427580,3408604644,4176884456}return SynapseXen_iIilIIIlllIIiIiiI[3323382624]end)("iIiilillili","lIliiiilIllIli",{},{},{}))then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2636291428]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="what are you trying to say? that fucking one dot + dot + dot + many dots is not adding adding 1 dot + dot and then adding all the dots together????"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2270146748,2817448326)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1091749430,1091754788)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2636291428]=SynapseXen_lillIIili(SynapseXen_lillIIili(2842951983,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(99826563,SynapseXen_IIliiIliIl[13]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3207669310}return SynapseXen_iIilIIIlllIIiIiiI[2636291428]end)("iiIiIi",14663,"IiiIlliIiIIIiIillli",{})),SynapseXen_ilIIiIIIllIli,256)local SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[1514833571]or(function()local SynapseXen_IlIlllllIIIIlIilii="yed"SynapseXen_iIilIIIlllIIiIiiI[1514833571]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(810968983,1160511776),SynapseXen_lillIIili(2104222600,SynapseXen_IIliiIliIl[12]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3821218837,2980962536,2720779423,493965678,2596239300,115492950,1605769109,2609828450,838389198}return SynapseXen_iIilIIIlllIIiIiiI[1514833571]end)()),SynapseXen_ilIIiIIIllIli,512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;local SynapseXen_lIIliiI=SynapseXen_lllIIiiIIIlIiIilii+2;local SynapseXen_IlIIIliIilliilIi={SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii](SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+1],SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+2])}for SynapseXen_llllIiIIlliIilIlIl=1,SynapseXen_lllil do SynapseXen_IliliIlilllliIIlll[SynapseXen_lIIliiI+SynapseXen_llllIiIIlliIilIlIl]=SynapseXen_IlIIIliIilliilIi[SynapseXen_llllIiIIlliIilIlIl]end;if SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+3]~=nil then SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+2]=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+3]else SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[582747307]or(function()local SynapseXen_IlIlllllIIIIlIilii="my way to go against expwoiting is to have safety measuwes. i 1 wocawscwipt and onwy moduwes. hewe's how it wowks: this scwipt bewow stowes the moduwes in a tabwe fow each moduwe we send the wist with the moduwes and moduwe infowmation and use inyit a function in my moduwe that wiww stowe the info and aftew it has send to aww the moduwes it wiww dewete them. so whenyevew the cwient twies to hack they cant get the moduwes. onwy this peace of wocawscwipt."SynapseXen_iIilIIIlllIIiIiiI[582747307]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3186171511,1249802084),SynapseXen_lillIIili(3994345531,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3475203112,3636505974,1028568718,744484078}return SynapseXen_iIilIIIlllIIiIiiI[582747307]end)())then SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3215801657]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="aspect network better obfuscator"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1187005826,2934876530)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1685407267,1685475892)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3215801657]=SynapseXen_lillIIili(SynapseXen_lillIIili(902393444,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3534414803,SynapseXen_IIliiIliIl[11]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2013218501,1132239633,360653943}return SynapseXen_iIilIIIlllIIiIiiI[3215801657]end)("iIIiI","l","IIIIllllIllliilIll",13991,"II","iIIiiiIl","II","llIIiIIiIlI"))]=SynapseXen_lIlIiiIiIi[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[1490906989]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i'm intercommunication about the most nonecclesiastical dll exploits for esp. they only characterization objects with a antepatriarchal in the geistesgeschichte for the esp."local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(830573,635323111)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2116010623,2116016341)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1490906989]=SynapseXen_lillIIili(SynapseXen_lillIIili(3588228925,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1502595879,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1106113210,2213262800,2008256359,2588576818,1876824609,4170372155,2725113000,3488228180}return SynapseXen_iIilIIIlllIIiIiiI[1490906989]end)(5642,4322,{},4064,8555,"lllIiI",1709,{},"liIlIllIilIIlilIlll"))]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3602171236]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi xen doesn't work on sk8r please help"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1161788138,3884705540)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3722853718,3722843813)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3602171236]=SynapseXen_lillIIili(SynapseXen_lillIIili(1907539383,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1934624722,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1155241165,2200952290,3316238324,982748623}return SynapseXen_iIilIIIlllIIiIiiI[3602171236]end)({},{}))then SynapseXen_lIlIiiIiIi[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[67970848]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="yed"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2424588852,1914421032)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1890687459,1890746023)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[67970848]=SynapseXen_lillIIili(SynapseXen_lillIIili(193245667,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1084312964,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2475922042,1060174693,772678166}return SynapseXen_iIilIIIlllIIiIiiI[67970848]end)("Ii","IilllIi"),512)]=SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3527301812]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="skisploit is the superior obfuscator, clearly."local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3688731992,3041248805)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1391147196,1391134515)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3527301812]=SynapseXen_lillIIili(SynapseXen_lillIIili(665631850,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1354125661,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3469236998,530603713,406071984,3193590438}return SynapseXen_iIilIIIlllIIiIiiI[3527301812]end)("iliIiliiIili",{},{},{},"IllIlililiIilliIlil",2848,13569,"lllllllIiIIiIIli"))]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[4290032619]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3718316195,181985047)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1155645606,1155706417)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4290032619]=SynapseXen_lillIIili(SynapseXen_lillIIili(3723862104,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3889771866,SynapseXen_IIliiIliIl[3]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1532391290,2263579254,2888166956,724063910,2563649356,718308909}return SynapseXen_iIilIIIlllIIiIiiI[4290032619]end)(6836,"I",{}))then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2339221661]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="what are you trying to say? that fucking one dot + dot + dot + many dots is not adding adding 1 dot + dot and then adding all the dots together????"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3553243390,476729831)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(625644557,625709846)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2339221661]=SynapseXen_lillIIili(SynapseXen_lillIIili(3376069444,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2762082229,SynapseXen_IIliiIliIl[4]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{281266306,3721393341,3908704175,1618776047,885190767,2259345933,2189233825,2757722062}return SynapseXen_iIilIIIlllIIiIiiI[2339221661]end)({},"iIil",6925,{},{},{},{},{}),256)local SynapseXen_lIlIlIiIliiIIlIli={}for SynapseXen_llllIiIIlliIilIlIl=1,#SynapseXen_IilIIliiiliII do local SynapseXen_llliiIillI=SynapseXen_IilIIliiiliII[SynapseXen_llllIiIIlliIilIlIl]for SynapseXen_lIlIIilIillli=0,#SynapseXen_llliiIillI do local SynapseXen_lilIIlliI=SynapseXen_llliiIillI[SynapseXen_lIlIIilIillli]local SynapseXen_lliilillIl=SynapseXen_lilIIlliI[1]local SynapseXen_IIiiillIiIiIil=SynapseXen_lilIIlliI[2]if SynapseXen_lliilillIl==SynapseXen_IliliIlilllliIIlll and SynapseXen_IIiiillIiIiIil>=SynapseXen_lllIIiiIIIlIiIilii then SynapseXen_lIlIlIiIliiIIlIli[SynapseXen_IIiiillIiIiIil]=SynapseXen_lliilillIl[SynapseXen_IIiiillIiIiIil]SynapseXen_lilIIlliI[1]=SynapseXen_lIlIlIiIliiIIlIli end end end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2986607984]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi my 2.5mb script doesn't work with xen please help"SynapseXen_iIilIIIlllIIiIiiI[2986607984]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3953794399,2012982195),SynapseXen_lillIIili(904775121,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4025173989,3989514298,2106629895,366231159,1651099217,2645637152,3264150180,2125323712}return SynapseXen_iIilIIIlllIIiIiiI[2986607984]end)())then local SynapseXen_IIlill=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3521544600]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="pain exist is gonna connect the dots of xen"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2178788949,1062438022)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(869695756,869724454)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3521544600]=SynapseXen_lillIIili(SynapseXen_lillIIili(2480213383,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2229626814,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1699859560,3823832586,3735459501,1717181366,675338706,3867764819,1542300639}return SynapseXen_iIilIIIlllIIiIiiI[3521544600]end)(2500,693),512)local SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[90826055]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="epic gamer vision"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2388369131,1054677408)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(4054312855,4054314882)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[90826055]=SynapseXen_lillIIili(SynapseXen_lillIIili(369915696,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1064597580,SynapseXen_IIliiIliIl[6]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{338016755,73030414,3534434840,484949236,2757909427}return SynapseXen_iIilIIIlllIIiIiiI[90826055]end)("ililliIiiii","iIil","iIIliIliIIiIiiIiIIi",4181),512),SynapseXen_ilIIiIIIllIli,512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3587241145]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="SECURE API, IMPOSSIBLE TO BYPASS!"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3901700274,2456204333)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2709611275,2709675281)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3587241145]=SynapseXen_lillIIili(SynapseXen_lillIIili(1315743938,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2637749942,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3517916648,3682355971,1382011631,2987952311,830056876,313995853}return SynapseXen_iIilIIIlllIIiIiiI[3587241145]end)({},"lil",{},{},{},"l","lIiliillIiiilIII","iiiIiiIIIlIlIli",{})),SynapseXen_ilIIiIIIllIli,256)]=SynapseXen_IIlill%SynapseXen_lllil elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[1320931762]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i put more time into this shitty list of dead memes then i did into the obfuscator itself"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2324494214,1363872577)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3287065612,3287130091)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1320931762]=SynapseXen_lillIIili(SynapseXen_lillIIili(264345907,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2109359623,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{99219326,2624801728}return SynapseXen_iIilIIIlllIIiIiiI[1320931762]end)({},{}))then SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[4291301317]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="aspect network better obfuscator"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2281870583,2802884872)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2051239888,2051305810)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4291301317]=SynapseXen_lillIIili(SynapseXen_lillIIili(481234439,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1001205625,SynapseXen_IIliiIliIl[12]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{16054455,2356038838}return SynapseXen_iIilIIIlllIIiIiiI[4291301317]end)({},"lllIiliiiIiIIii"))]=SynapseXen_iiillIIIlIi[SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[518845294],SynapseXen_iIilIIIlllIIiIiiI[1175631070]or(function()local SynapseXen_IlIlllllIIIIlIilii="luraph better then xen bros :pensive:"SynapseXen_iIilIIIlllIIiIiiI[1175631070]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(655334817,286288235),SynapseXen_lillIIili(2679953950,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3321317358,4153939552,4121972836,3609596250,1063084938,2545353568,950265001}return SynapseXen_iIilIIIlllIIiIiiI[1175631070]end)(),262144)]]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[381159685]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi devforum"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1755401632,4008970270)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3395595375,3395660475)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[381159685]=SynapseXen_lillIIili(SynapseXen_lillIIili(1393007980,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2003440835,SynapseXen_IIliiIliIl[4]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{810462678,1281407955,931577924}return SynapseXen_iIilIIIlllIIiIiiI[381159685]end)("IIIlIiIllIil","iliIIIi","IlIilllIllllI",8845,"liilIlI",8625,"IIlillliIllIlII"))then SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2596273221]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="print(bytecode)"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1256197158,4183562497)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3425113018,3425133561)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2596273221]=SynapseXen_lillIIili(SynapseXen_lillIIili(326828050,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(159009140,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1363043343}return SynapseXen_iIilIIIlllIIiIiiI[2596273221]end)({},"lllllIiililIlIl","lIlIIiiiIliIllliIll",{}))]=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[518845294],SynapseXen_iIilIIIlllIIiIiiI[1265559681]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i'm intercommunication about the most nonecclesiastical dll exploits for esp. they only characterization objects with a antepatriarchal in the geistesgeschichte for the esp."local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2439530957,4152556983)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2888109622,2888134234)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1265559681]=SynapseXen_lillIIili(SynapseXen_lillIIili(368107573,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1010449956,SynapseXen_IIliiIliIl[2]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{648319955,1347085362,3190109095,3753373888,4236569195,3688197835,1648155285,1486766172,3658741395}return SynapseXen_iIilIIIlllIIiIiiI[1265559681]end)(3229))]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3089838453]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen doesn't come with instance caching, sorry superskater"SynapseXen_iIilIIIlllIIiIiiI[3089838453]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3493076858,223848682),SynapseXen_lillIIili(1960831275,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{578006613,3463332497,3949691750,1562033692,272533857}return SynapseXen_iIilIIIlllIIiIiiI[3089838453]end)())then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2561871518]or(function()local SynapseXen_IlIlllllIIIIlIilii="sponsored by ironbrew, jk xen is better"SynapseXen_iIilIIIlllIIiIiiI[2561871518]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(4069142680,3990661646),SynapseXen_lillIIili(3068721243,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1301049285,1880864422,1915356887,25616702,4086431,1226192571,1733544949,370498661,3197545072,3704181853}return SynapseXen_iIilIIIlllIIiIiiI[2561871518]end)(),256)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]=assert(tonumber(SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]),'`for` initial value must be a number')SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+1]=assert(tonumber(SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+1]),'`for` limit must be a number')SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+2]=assert(tonumber(SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+2]),'`for` step must be a number')SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]-SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+2]SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+SynapseXen_iIIiillIiIIililIiiI[1050029882]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2043231385]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi xen crashes on my axon paste plz help"SynapseXen_iIilIIIlllIIiIiiI[2043231385]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(4256616072,327805463),SynapseXen_lillIIili(1955578998,SynapseXen_IIliiIliIl[8]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{29194442,3352980194,2346143736,2658277971,942871935,489885084,3078272641,1242808744,1459917082}return SynapseXen_iIilIIIlllIIiIiiI[2043231385]end)())then local SynapseXen_IIlill=SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[2196078565]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="pain exist is gonna connect the dots of xen"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(4245564312,2471159986)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(612363223,612362123)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2196078565]=SynapseXen_lillIIili(SynapseXen_lillIIili(160689732,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3320708643,SynapseXen_IIliiIliIl[4]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1265832481,2927977730,2365639142,667544432,59897269,2406687417}return SynapseXen_iIilIIIlllIIiIiiI[2196078565]end)({},2407,{},2171,"IiIIllii",13829,1230,"liliIlIiIillIllIii",12669))local SynapseXen_lllil=SynapseXen_iiIIlliIIIiliil(SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[4202666175]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="baby i just fell for uwu,,,,,, i wanna be with uwu!11!!"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3570436412,1922082290)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(704554637,704565488)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4202666175]=SynapseXen_lillIIili(SynapseXen_lillIIili(4024928716,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2066940837,SynapseXen_IIliiIliIl[1]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{828512,2398954616,1914566850,1299347199,739756143,1709600322}return SynapseXen_iIilIIIlllIIiIiiI[4202666175]end)("ii",{},"lIiIliill"),512),SynapseXen_ilIIiIIIllIli,512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[446171410]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i put more time into this shitty list of dead memes then i did into the obfuscator itself"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(392767067,3072270886)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(35916801,35921227)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[446171410]=SynapseXen_lillIIili(SynapseXen_lillIIili(1312381539,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1207028350,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{726608450}return SynapseXen_iIilIIIlllIIiIiiI[446171410]end)("IiIIIIIliiliIIIlII",3493,3574,"IliiiiIIliIIl",4608))]=SynapseXen_IIlill^SynapseXen_lllil elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3548750368]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="print(bytecode)"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1651299849,1216736918)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(4054443061,4054440624)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3548750368]=SynapseXen_lillIIili(SynapseXen_lillIIili(2690314877,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(588050040,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{541477839,1253632204,1947096263}return SynapseXen_iIilIIIlllIIiIiiI[3548750368]end)({},1699,{},{},{}))then SynapseXen_IliliIlilllliIIlll[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3489371525]or(function()local SynapseXen_IlIlllllIIIIlIilii="pain is gonna use the backspace method on xen"SynapseXen_iIilIIIlllIIiIiiI[3489371525]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1133630880,2059793982),SynapseXen_lillIIili(2695688506,SynapseXen_IIliiIliIl[6]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3244190792,199463128,2677743966,867696038,2566444952}return SynapseXen_iIilIIIlllIIiIiiI[3489371525]end)(),256)]=-SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[1510506967]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i'm intercommunication about the most nonecclesiastical dll exploits for esp. they only characterization objects with a antepatriarchal in the geistesgeschichte for the esp."local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2806499701,3374529122)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3752725930,3752722694)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1510506967]=SynapseXen_lillIIili(SynapseXen_lillIIili(3857889967,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2451188122,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{701379938,502769644,2944171149,4256014885,800575025,3795409961,3262643199,1711851139}return SynapseXen_iIilIIIlllIIiIiiI[1510506967]end)("lIlIIIIlIiiI",7570,{},5051),512)]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2826788529]or(function()local SynapseXen_IlIlllllIIIIlIilii="yed"SynapseXen_iIilIIIlllIIiIiiI[2826788529]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1370075006,4273121090),SynapseXen_lillIIili(111508265,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4129755025,785028090,2543701551,2904162126,2087494391}return SynapseXen_iIilIIIlllIIiIiiI[2826788529]end)())then SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1951151924]or(function()local SynapseXen_IlIlllllIIIIlIilii="luraph better then xen bros :pensive:"SynapseXen_iIilIIIlllIIiIiiI[1951151924]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(488883670,3930261593),SynapseXen_lillIIili(1591355764,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2113401070,1487445835,484206672,404270763,2993498032,1783297110,3495309294,951099653}return SynapseXen_iIilIIIlllIIiIiiI[1951151924]end)())]=SynapseXen_lillIIili(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[1426715205]or(function()local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"SynapseXen_iIilIIIlllIIiIiiI[1426715205]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2427876787,3029025707),SynapseXen_lillIIili(2248435893,SynapseXen_IIliiIliIl[4]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1337749185,2098699377,1345843335}return SynapseXen_iIilIIIlllIIiIiiI[1426715205]end)(),512),SynapseXen_ilIIiIIIllIli)~=0;if SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[3278063883]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi devforum"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1613458802,2110796031)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1232138383,1232137196)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3278063883]=SynapseXen_lillIIili(SynapseXen_lillIIili(3838994383,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3769397219,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1324326729,2039119138,2657822635,3142900109,3224689325,4018440766}return SynapseXen_iIilIIIlllIIiIiiI[3278063883]end)({},7086,"liliIiIliIlilllil","llIillil",{},{},{},{}),512)~=0 then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2933344463]or(function()local SynapseXen_IlIlllllIIIIlIilii="what are you trying to say? that fucking one dot + dot + dot + many dots is not adding adding 1 dot + dot and then adding all the dots together????"SynapseXen_iIilIIIlllIIiIiiI[2933344463]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3662256307,1329009903),SynapseXen_lillIIili(2365489392,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3391461199,2595140696,3942372723,3879841043,4149502203,4230252685,3591681841,3685087336}return SynapseXen_iIilIIIlllIIiIiiI[2933344463]end)())then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2359185010]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="sponsored by ironbrew, jk xen is better"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1123605042,2820916084)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1026859052,1026927804)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2359185010]=SynapseXen_lillIIili(SynapseXen_lillIIili(2538547952,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3559794865,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3897024652,4191163986,1797164006,153719437,771210646}return SynapseXen_iIilIIIlllIIiIiiI[2359185010]end)(12260,{},"llIIliIiilIII",11653,"llIilIIl","li",11331,"lliiiIiI"))local SynapseXen_IIlill=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[2682067868]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="HELP ME PEOPLE ARE CRASHING MY GAME PLZ HELP"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(71770006,468785193)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3009071718,3009078276)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2682067868]=SynapseXen_lillIIili(SynapseXen_lillIIili(2470596607,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(625620536,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{333630871,102108185,2053934838,313006634,3434386885,4172697692,4116712399,3302445977,3710748945,2973916327}return SynapseXen_iIilIIIlllIIiIiiI[2682067868]end)("ll",{},{},{},2905,{},"ilIillIIlIiIiI",14501),512)local SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[1860822674]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="https://twitter.com/Ripull_RBLX/status/1059334518581145603"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3808483394,1901402246)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2303840862,2303835540)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1860822674]=SynapseXen_lillIIili(SynapseXen_lillIIili(573547093,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1011004850,SynapseXen_IIliiIliIl[13]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1473228531,2744641641,321441416,777945064,4228792234,3616522581,3995800985}return SynapseXen_iIilIIIlllIIiIiiI[1860822674]end)(7000,{},"IIiiiiII",10989,"iIIIIlillIllil",11833,9936,"IIiillliiIIllIii","iIilllIIi",9084)),SynapseXen_ilIIiIIIllIli,512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+1]=SynapseXen_IIlill;SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]=SynapseXen_IIlill[SynapseXen_lllil]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2853491470]or(function()local SynapseXen_IlIlllllIIIIlIilii="inb4 posted on exploit reports section"SynapseXen_iIilIIIlllIIiIiiI[2853491470]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(983300478,3755336257),SynapseXen_lillIIili(1291277937,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1234603512,1639839436,2025780192,2078814489,1833387705,1083901039,545940624}return SynapseXen_iIilIIIlllIIiIiiI[2853491470]end)())then SynapseXen_iiillIIIlIi[SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[518845294],SynapseXen_iIilIIIlllIIiIiiI[3739461669]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen doesn't come with instance caching, sorry superskater"SynapseXen_iIilIIIlllIIiIiiI[3739461669]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3915052684,165601283),SynapseXen_lillIIili(2915904559,SynapseXen_IIliiIliIl[9]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2887540590,3865907328,824362566,8425882,2615038291,1662005705,3268807862}return SynapseXen_iIilIIIlllIIiIiiI[3739461669]end)())]]=SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1810394279]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="epic gamer vision"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1723256075,65819536)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1949660421,1949720434)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1810394279]=SynapseXen_lillIIili(SynapseXen_lillIIili(767330358,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1202445607,SynapseXen_IIliiIliIl[11]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3775078986}return SynapseXen_iIilIIIlllIIiIiiI[1810394279]end)("iI",13695,"iliiiIIlIiiliIlIi",2955,"iIIlllllillII",14806,5814,9864,6207))]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3437581789]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="xen best rerubi paste"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2250343124,582897329)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3850991880,3851048237)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3437581789]=SynapseXen_lillIIili(SynapseXen_lillIIili(456251280,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3082681843,SynapseXen_IIliiIliIl[12]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2100572693,3325412104,1461771846,2158294164}return SynapseXen_iIilIIIlllIIiIiiI[3437581789]end)(8506,{},10578,{},{},2141))then local SynapseXen_IIlill=SynapseXen_lillIIili(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3058520605]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="can we have an f in chat for ripull"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(999766184,1399777992)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(4071305662,4071297136)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3058520605]=SynapseXen_lillIIili(SynapseXen_lillIIili(1058478559,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(4266801384,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1965231743,2288511081,4152498704,3493861201,908748842,464036334}return SynapseXen_iIilIIIlllIIiIiiI[3058520605]end)("lIllii","IliilllIilllillIl","lIll"),512),SynapseXen_ilIIiIIIllIli)local SynapseXen_lllil=SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[3418751402]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="wait for someone on devforum to say they are gonna deobfuscate this"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3976656465,806590631)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3190872962,3190876018)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3418751402]=SynapseXen_lillIIili(SynapseXen_lillIIili(3206341221,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3417688964,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3204546076,702775037,138111363,2181821498}return SynapseXen_iIilIIIlllIIiIiiI[3418751402]end)("ill",{},5425,12886))local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[386885406]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi my 2.5mb script doesn't work with xen please help"SynapseXen_iIilIIIlllIIiIiiI[386885406]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1573898458,3223410401),SynapseXen_lillIIili(877512065,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2807436180,1517762326,107407599,74101615,866612971,2553550895,3907656633}return SynapseXen_iIilIIIlllIIiIiiI[386885406]end)())]=SynapseXen_IIlill+SynapseXen_lllil elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[4193448928]or(function()local SynapseXen_IlIlllllIIIIlIilii="this is a christian obfuscator, no cursing allowed in our scripts"SynapseXen_iIilIIIlllIIiIiiI[4193448928]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3562138674,2940088949),SynapseXen_lillIIili(3536924126,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3145521844,1295369613,2344452272,482533090,759320935,413464573}return SynapseXen_iIilIIIlllIIiIiiI[4193448928]end)())then SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1050114435]or(function()local SynapseXen_IlIlllllIIIIlIilii="what are you trying to say? that fucking one dot + dot + dot + many dots is not adding adding 1 dot + dot and then adding all the dots together????"SynapseXen_iIilIIIlllIIiIiiI[1050114435]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3363459565,2376040076),SynapseXen_lillIIili(3965260767,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3235909624,3932742327,34118740,3426403951,1457573383,1523318706,114110743,2997724041,3786227955,2508753550}return SynapseXen_iIilIIIlllIIiIiiI[1050114435]end)())]=SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3748871018]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi xen doesn't work on sk8r please help"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1471683466,3802208187)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3793474325,3793480829)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3748871018]=SynapseXen_lillIIili(SynapseXen_lillIIili(3154791159,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(998644781,SynapseXen_IIliiIliIl[1]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4268199960}return SynapseXen_iIilIIIlllIIiIiiI[3748871018]end)({},{},4210,{}))]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[597866468]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="level 1 crook = luraph, level 100 boss = xen"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1454666202,883512418)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(867064137,867123523)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[597866468]=SynapseXen_lillIIili(SynapseXen_lillIIili(3581852094,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3211999465,SynapseXen_IIliiIliIl[12]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3332398400,2352512636,505003558,3328057132,1178169705,84797142,4118679884,662311643,1282218847,3228273305}return SynapseXen_iIilIIIlllIIiIiiI[597866468]end)({},13456,"ilIi","IIiIiiIiiIiIIlIiI"))then SynapseXen_IliliIlilllliIIlll[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1657839689]or(function()local SynapseXen_IlIlllllIIIIlIilii="my way to go against expwoiting is to have safety measuwes. i 1 wocawscwipt and onwy moduwes. hewe's how it wowks: this scwipt bewow stowes the moduwes in a tabwe fow each moduwe we send the wist with the moduwes and moduwe infowmation and use inyit a function in my moduwe that wiww stowe the info and aftew it has send to aww the moduwes it wiww dewete them. so whenyevew the cwient twies to hack they cant get the moduwes. onwy this peace of wocawscwipt."SynapseXen_iIilIIIlllIIiIiiI[1657839689]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2919462347,1627458346),SynapseXen_lillIIili(2150104950,SynapseXen_IIliiIliIl[2]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1270855167}return SynapseXen_iIilIIIlllIIiIiiI[1657839689]end)(),256)]=#SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3708134369]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="baby i just fell for uwu,,,,,, i wanna be with uwu!11!!"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(96631563,269340992)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2227531972,2227597258)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3708134369]=SynapseXen_lillIIili(SynapseXen_lillIIili(3353950077,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2073594144,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2408674731,1871505551,3184364591,279518663,3624943521,1529864737,3308814268,2792110191,381324444}return SynapseXen_iIilIIIlllIIiIiiI[3708134369]end)({},"l",3062)),SynapseXen_ilIIiIIIllIli,512)]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[559887249]or(function()local SynapseXen_IlIlllllIIIIlIilii="pain exist is gonna connect the dots of xen"SynapseXen_iIilIIIlllIIiIiiI[559887249]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2583213148,798900549),SynapseXen_lillIIili(2951071342,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1211209173}return SynapseXen_iIilIIIlllIIiIiiI[559887249]end)())then local SynapseXen_IIlill=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[1465465782]or(function()local SynapseXen_IlIlllllIIIIlIilii="SYNAPSE XEN [FE BYPASS] [BETTER THEN LURAPH] [AMAZING] OMG OMG OMG !!!!!!"SynapseXen_iIilIIIlllIIiIiiI[1465465782]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3026176846,4137575513),SynapseXen_lillIIili(3946895914,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{159189245,1040213373,2517283157,2745746878,3317486282,3079260363,397291688,3097972602,2389118263,2507409515}return SynapseXen_iIilIIIlllIIiIiiI[1465465782]end)(),512)local SynapseXen_lllil=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[63729157]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i put more time into this shitty list of dead memes then i did into the obfuscator itself"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2083714173,1745524105)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(612625411,612612869)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[63729157]=SynapseXen_lillIIili(SynapseXen_lillIIili(3170959658,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(9109561,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4096121931,304453427,3224480283,1026950180,4015180769,3288068072}return SynapseXen_iIilIIIlllIIiIiiI[63729157]end)({},7103,{},{},"iliIIIiiIIl",14131,9825,6419,{}),512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1016875277]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="wally bad bird"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2722567521,811714572)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1310849994,1310881117)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1016875277]=SynapseXen_lillIIili(SynapseXen_lillIIili(2495664341,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2941841429,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{825566357,1210174112,2069895848,1133867045,2289850212,4208481353,2853304913}return SynapseXen_iIilIIIlllIIiIiiI[1016875277]end)({}),256)]=SynapseXen_IIlill*SynapseXen_lllil elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[4271217374]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(4168273750,1652180957)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2864749858,2864821130)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4271217374]=SynapseXen_lillIIili(SynapseXen_lillIIili(162468023,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(974869102,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4088129395,1071787939,491525886,187238322,2463193177,467263512,2550618164,3168785305,2393017726}return SynapseXen_iIilIIIlllIIiIiiI[4271217374]end)(1340,4616,{},13262,1729,{}))then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_iiIIlliIIIiliil(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3807806416]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="xen detects custom getfenv"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1685306228,1865356442)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1381896583,1381889734)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3807806416]=SynapseXen_lillIIili(SynapseXen_lillIIili(1072733001,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(757820056,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{195764898,2806027459,1136016836,2007415545,841507473}return SynapseXen_iIilIIIlllIIiIiiI[3807806416]end)(14890,{},915,4764,"iIIIlIlllI",7633,{},13026,{}),256),SynapseXen_ilIIiIIIllIli,256)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;local SynapseXen_iIlliliiilliIlI=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+2]local SynapseXen_iIllIlliIi=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]+SynapseXen_iIlliliiilliIlI;SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]=SynapseXen_iIllIlliIi;if SynapseXen_iIlliliiilliIlI>0 then if SynapseXen_iIllIlliIi<=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+1]then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+SynapseXen_iIIiillIiIIililIiiI[1050029882]SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+3]=SynapseXen_iIllIlliIi end else if SynapseXen_iIllIlliIi>=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+1]then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+SynapseXen_iIIiillIiIIililIiiI[1050029882]SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+3]=SynapseXen_iIllIlliIi end end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3379972723]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="xen doesn't come with instance caching, sorry superskater"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(693250496,4126739776)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2993885897,2993894589)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3379972723]=SynapseXen_lillIIili(SynapseXen_lillIIili(4026377260,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2599157628,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2203975291}return SynapseXen_iIilIIIlllIIiIiiI[3379972723]end)(4417))then local SynapseXen_IIlill=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[479875370]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="epic gamer vision"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3415807130,3218167084)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2826050974,2826054091)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[479875370]=SynapseXen_lillIIili(SynapseXen_lillIIili(3825700701,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(972213173,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3960666960,1293003642,2734523052,417544296,1839950856,3086850787,1296537713}return SynapseXen_iIilIIIlllIIiIiiI[479875370]end)({},"lIiIiiIl"),512)local SynapseXen_lllil=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[2223041088]or(function()local SynapseXen_IlIlllllIIIIlIilii="so if you'we nyot awawe of expwoiting by this point, you've pwobabwy been wiving undew a wock that the pionyeews used to wide fow miwes. wobwox is often seen as an expwoit-infested gwound by most fwom the suwface, awthough this isn't the case."SynapseXen_iIilIIIlllIIiIiiI[2223041088]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(480646863,577499318),SynapseXen_lillIIili(2541165336,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{55183742,4053955912}return SynapseXen_iIilIIIlllIIiIiiI[2223041088]end)(),512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2234721722]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="https://twitter.com/Ripull_RBLX/status/1059334518581145603"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3570269008,1093776060)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1365667504,1365719092)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2234721722]=SynapseXen_lillIIili(SynapseXen_lillIIili(888536900,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(146210240,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3674698808}return SynapseXen_iIilIIIlllIIiIiiI[2234721722]end)({},2946,"IIIllIlilIIiIIlli",5073,"iIiiIliiIiiIiiii","llllIlilliiili"))]=SynapseXen_IIlill/SynapseXen_lllil elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[1376549284]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="sponsored by ironbrew, jk xen is better"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1110547624,2816673874)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2257007081,2257011886)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1376549284]=SynapseXen_lillIIili(SynapseXen_lillIIili(3128897021,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(4128385720,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3843922684,2706399535,1359112290,4228282943}return SynapseXen_iIilIIIlllIIiIiiI[1376549284]end)({},{},{},2889,"iliI","IiIIlIillIll",{},"liIIIiliIIIlIlllll"))then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[4097270858]or(function()local SynapseXen_IlIlllllIIIIlIilii="SYNAPSE XEN [FE BYPASS] [BETTER THEN LURAPH] [AMAZING] OMG OMG OMG !!!!!!"SynapseXen_iIilIIIlllIIiIiiI[4097270858]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2909967071,339921584),SynapseXen_lillIIili(278051829,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{483216289,1072774428,147676610,4015583445,206578269}return SynapseXen_iIilIIIlllIIiIiiI[4097270858]end)(),256)local SynapseXen_IIlill=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[1614021739]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="print(bytecode)"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3232312524,2133327397)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1427045275,1427041573)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1614021739]=SynapseXen_lillIIili(SynapseXen_lillIIili(4285451317,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1753089230,SynapseXen_IIliiIliIl[5]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{989665073,2766311298,1217386923}return SynapseXen_iIilIIIlllIIiIiiI[1614021739]end)("ili","Iiii",{},{},{}),512)local SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[273684473]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi xen doesn't work on sk8r please help"SynapseXen_iIilIIIlllIIiIiiI[273684473]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1547192495,4205096816),SynapseXen_lillIIili(253758056,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{176608871,2875963927,1319367961,2679045549,2628663847,2899300208}return SynapseXen_iIilIIIlllIIiIiiI[273684473]end)(),512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;local SynapseXen_ilIlliiIIlIil,SynapseXen_IIIliIl;local SynapseXen_lIiIiliIiliiillIll,SynapseXen_iIlllIilIIII;SynapseXen_ilIlliiIIlIil={}if SynapseXen_IIlill~=1 then if SynapseXen_IIlill~=0 then SynapseXen_lIiIiliIiliiillIll=SynapseXen_lllIIiiIIIlIiIilii+SynapseXen_IIlill-1 else SynapseXen_lIiIiliIiliiillIll=SynapseXen_lllilIiIill end;SynapseXen_iIlllIilIIII=0;for SynapseXen_llllIiIIlliIilIlIl=SynapseXen_lllIIiiIIIlIiIilii+1,SynapseXen_lIiIiliIiliiillIll do SynapseXen_iIlllIilIIII=SynapseXen_iIlllIilIIII+1;SynapseXen_ilIlliiIIlIil[SynapseXen_iIlllIilIIII]=SynapseXen_lliilillIl[SynapseXen_llllIiIIlliIilIlIl]end;SynapseXen_lIiIiliIiliiillIll,SynapseXen_IIIliIl=SynapseXen_IillIiIlIlIlIlill(SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii](unpack(SynapseXen_ilIlliiIIlIil,1,SynapseXen_lIiIiliIiliiillIll-SynapseXen_lllIIiiIIIlIiIilii)))else SynapseXen_lIiIiliIiliiillIll,SynapseXen_IIIliIl=SynapseXen_IillIiIlIlIlIlill(SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]())end;SynapseXen_lllilIiIill=SynapseXen_lllIIiiIIIlIiIilii-1;if SynapseXen_lllil~=1 then if SynapseXen_lllil~=0 then SynapseXen_lIiIiliIiliiillIll=SynapseXen_lllIIiiIIIlIiIilii+SynapseXen_lllil-2 else SynapseXen_lIiIiliIiliiillIll=SynapseXen_lIiIiliIiliiillIll+SynapseXen_lllIIiiIIIlIiIilii-1 end;SynapseXen_iIlllIilIIII=0;for SynapseXen_llllIiIIlliIilIlIl=SynapseXen_lllIIiiIIIlIiIilii,SynapseXen_lIiIiliIiliiillIll do SynapseXen_iIlllIilIIII=SynapseXen_iIlllIilIIII+1;SynapseXen_lliilillIl[SynapseXen_llllIiIIlliIilIlIl]=SynapseXen_IIIliIl[SynapseXen_iIlllIilIIII]end end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2687546875]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1094439595,749887476)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2037171684,2037175146)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2687546875]=SynapseXen_lillIIili(SynapseXen_lillIIili(1694986855,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2704761388,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3051600361,952271951,637162109,3610797179,487475730,2384274303,942027125,2702797043,2577915003,556238327}return SynapseXen_iIilIIIlllIIiIiiI[2687546875]end)("Ili","lIIIiI","lIlilliilIllII",{},10073,"lllllliIIIIllIIlIIl",4101))then if not not SynapseXen_IliliIlilllliIIlll[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1114412471]or(function()local SynapseXen_IlIlllllIIIIlIilii="wait for someone on devforum to say they are gonna deobfuscate this"SynapseXen_iIilIIIlllIIiIiiI[1114412471]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(478192620,147956530),SynapseXen_lillIIili(1494993305,SynapseXen_IIliiIliIl[9]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3099078764,3672013963,4142360211,2606625304,2411298188,4240642582,3147385934}return SynapseXen_iIilIIIlllIIiIiiI[1114412471]end)(),256)]==(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[712660855]or(function()local SynapseXen_IlIlllllIIIIlIilii="pain exist is gonna connect the dots of xen"SynapseXen_iIilIIIlllIIiIiiI[712660855]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1838931641,1904109281),SynapseXen_lillIIili(330139382,SynapseXen_IIliiIliIl[11]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1234974890,3750732520,2857544655,3011740063,3890183984,3093645205}return SynapseXen_iIilIIIlllIIiIiiI[712660855]end)(),512)==0)then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2494222979]or(function()local SynapseXen_IlIlllllIIIIlIilii="HELP ME PEOPLE ARE CRASHING MY GAME PLZ HELP"SynapseXen_iIilIIIlllIIiIiiI[2494222979]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1658817507,1655728202),SynapseXen_lillIIili(2598504958,SynapseXen_IIliiIliIl[8]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3038275040,2645060891,446998696,1107168202,682236466,3756189849,4163552818,1705605427,4061881597}return SynapseXen_iIilIIIlllIIiIiiI[2494222979]end)())then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3933836440]or(function()local SynapseXen_IlIlllllIIIIlIilii="skisploit is the superior obfuscator, clearly."SynapseXen_iIilIIIlllIIiIiiI[3933836440]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1902732946,28895647),SynapseXen_lillIIili(1480788969,SynapseXen_IIliiIliIl[5]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2062284412,2821721155,404135763,962531443}return SynapseXen_iIilIIIlllIIiIiiI[3933836440]end)())~=0;local SynapseXen_IIlill=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[208213297]or(function()local SynapseXen_IlIlllllIIIIlIilii="luraph better then xen bros :pensive:"SynapseXen_iIilIIIlllIIiIiiI[208213297]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2709891160,950611473),SynapseXen_lillIIili(3591212137,SynapseXen_IIliiIliIl[2]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{424633623,83263680,3877109572,3077224805,1908528290,1818141348,3611178928,2517520790}return SynapseXen_iIilIIIlllIIiIiiI[208213297]end)(),512)local SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[1418732007]or(function()local SynapseXen_IlIlllllIIIIlIilii="this is so sad, alexa play ripull.mp4"SynapseXen_iIilIIIlllIIiIiiI[1418732007]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(552095691,418969575),SynapseXen_lillIIili(1967390811,SynapseXen_IIliiIliIl[9]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2297534108,123184746,1498339922,379936744,239247789,1674040241,1939956074,268513196}return SynapseXen_iIilIIIlllIIiIiiI[1418732007]end)(),512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;if SynapseXen_IIlill==SynapseXen_lllil~=SynapseXen_lllIIiiIIIlIiIilii then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2638442793]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="level 1 crook = luraph, level 100 boss = xen"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2114822808,2007567141)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3993126757,3993188236)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2638442793]=SynapseXen_lillIIili(SynapseXen_lillIIili(1741531500,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1560223798,SynapseXen_IIliiIliIl[1]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2891073050,2116704662,3474683048,1599216558,4180921490,2871792525}return SynapseXen_iIilIIIlllIIiIiiI[2638442793]end)(5616,4345,14223,"iiillliliIIl",7649))then local SynapseXen_IIlill=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3145220437]or(function()local SynapseXen_IlIlllllIIIIlIilii="pain is gonna use the backspace method on xen"SynapseXen_iIilIIIlllIIiIiiI[3145220437]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3759597628,2248150896),SynapseXen_lillIIili(3484027479,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3932568881,2152958198,86243818}return SynapseXen_iIilIIIlllIIiIiiI[3145220437]end)(),512)local SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[577961911]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="thats how mafia works"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(965896077,2272767351)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1897089327,1897159256)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[577961911]=SynapseXen_lillIIili(SynapseXen_lillIIili(3775159898,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1836424394,SynapseXen_IIliiIliIl[1]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{450906811,4128251550,1508122823,4186490982,1481297707,313587762,1639575523,573888774,1741840106,151267317}return SynapseXen_iIilIIIlllIIiIiiI[577961911]end)(7414),512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_iiIIlliIIIiliil(SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2775759537]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen detects custom getfenv"SynapseXen_iIilIIIlllIIiIiiI[2775759537]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3841917544,2500645571),SynapseXen_lillIIili(3946899623,SynapseXen_IIliiIliIl[8]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2452928154,2044186583,2853052892}return SynapseXen_iIilIIIlllIIiIiiI[2775759537]end)(),256),SynapseXen_ilIIiIIIllIli,256)]=SynapseXen_IIlill-SynapseXen_lllil elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[199037498]or(function()local SynapseXen_IlIlllllIIIIlIilii="i'm intercommunication about the most nonecclesiastical dll exploits for esp. they only characterization objects with a antepatriarchal in the geistesgeschichte for the esp."SynapseXen_iIilIIIlllIIiIiiI[199037498]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(421198777,251298184),SynapseXen_lillIIili(3193802050,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1699747346,1469613155,2732952598,2430864477}return SynapseXen_iIilIIIlllIIiIiiI[199037498]end)())then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[4175061775]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi devforum"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1663766491,303043342)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(611385963,611393919)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4175061775]=SynapseXen_lillIIili(SynapseXen_lillIIili(2545393573,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1328547772,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{936666347,633928199,1029289180,1617945175,4205634400,3717468132,771568038,4276424150}return SynapseXen_iIilIIIlllIIiIiiI[4175061775]end)("IiliiilIliillII",10529,"iIlIIllIiIlIliIIl"),256)local SynapseXen_IIlill=SynapseXen_iiIIlliIIIiliil(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[1135556608]or(function()local SynapseXen_IlIlllllIIIIlIilii="pain is gonna use the backspace method on xen"SynapseXen_iIilIIIlllIIiIiiI[1135556608]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(4165772717,771437851),SynapseXen_lillIIili(1280237606,SynapseXen_IIliiIliIl[6]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{968539616,897127064,660845016,3321539544,3394882024,990784168,2471519352,1509191293,1475747984,2194543472}return SynapseXen_iIilIIIlllIIiIiiI[1135556608]end)()),SynapseXen_ilIIiIIIllIli,512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;local SynapseXen_ilIlliiIIlIil,SynapseXen_IIIliIl;local SynapseXen_lIiIiliIiliiillIll;local SynapseXen_lIlIllIiIIIlI=0;SynapseXen_ilIlliiIIlIil={}if SynapseXen_IIlill~=1 then if SynapseXen_IIlill~=0 then SynapseXen_lIiIiliIiliiillIll=SynapseXen_lllIIiiIIIlIiIilii+SynapseXen_IIlill-1 else SynapseXen_lIiIiliIiliiillIll=SynapseXen_lllilIiIill end;for SynapseXen_llllIiIIlliIilIlIl=SynapseXen_lllIIiiIIIlIiIilii+1,SynapseXen_lIiIiliIiliiillIll do SynapseXen_ilIlliiIIlIil[#SynapseXen_ilIlliiIIlIil+1]=SynapseXen_lliilillIl[SynapseXen_llllIiIIlliIilIlIl]end;SynapseXen_IIIliIl={SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii](unpack(SynapseXen_ilIlliiIIlIil,1,SynapseXen_lIiIiliIiliiillIll-SynapseXen_lllIIiiIIIlIiIilii))}else SynapseXen_IIIliIl={SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]()}end;for SynapseXen_iIllIlliIi in next,SynapseXen_IIIliIl do if SynapseXen_iIllIlliIi>SynapseXen_lIlIllIiIIIlI then SynapseXen_lIlIllIiIIIlI=SynapseXen_iIllIlliIi end end;return SynapseXen_IIIliIl,SynapseXen_lIlIllIiIIIlI elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2333450188]or(function()local SynapseXen_IlIlllllIIIIlIilii="imagine using some lua minifier tool and thinking you are a badass"SynapseXen_iIilIIIlllIIiIiiI[2333450188]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2934974031,2173584856),SynapseXen_lillIIili(2260956770,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{254404147}return SynapseXen_iIilIIIlllIIiIiiI[2333450188]end)())then if SynapseXen_lillIIili(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[518845294],SynapseXen_iIilIIIlllIIiIiiI[340812766]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen best rerubi paste"SynapseXen_iIilIIIlllIIiIiiI[340812766]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(691533760,1908506525),SynapseXen_lillIIili(4047871218,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3326363781,3286290788,1940607795}return SynapseXen_iIilIIIlllIIiIiiI[340812766]end)(),262144),SynapseXen_ilIIiIIIllIli)==(SynapseXen_iIilIIIlllIIiIiiI[1844891478]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen best rerubi paste"SynapseXen_iIilIIIlllIIiIiiI[1844891478]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(4243099828,1830975752),SynapseXen_lillIIili(2740885979,SynapseXen_IIliiIliIl[1]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2883091528,723970769,4023520677,571481363,2159332421,1008664819}return SynapseXen_iIilIIIlllIIiIiiI[1844891478]end)())then SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3218828697]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i put more time into this shitty list of dead memes then i did into the obfuscator itself"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(524342499,1746533317)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(4149785987,4149840496)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3218828697]=SynapseXen_lillIIili(SynapseXen_lillIIili(3372225622,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2529865328,SynapseXen_IIliiIliIl[5]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3700561752,2531082186,136122769,1899306034,221774493,1976847354,302337299,3282271068}return SynapseXen_iIilIIIlllIIiIiiI[3218828697]end)("liIIiilIlIII","IIlIiIlllIlIiili",{}),256)]=SynapseXen_IliIIlIIlIlIlI else SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3218828697]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i put more time into this shitty list of dead memes then i did into the obfuscator itself"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(524342499,1746533317)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(4149785987,4149840496)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3218828697]=SynapseXen_lillIIili(SynapseXen_lillIIili(3372225622,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2529865328,SynapseXen_IIliiIliIl[5]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3700561752,2531082186,136122769,1899306034,221774493,1976847354,302337299,3282271068}return SynapseXen_iIilIIIlllIIiIiiI[3218828697]end)("liIIiilIlIII","IIlIiIlllIlIiili",{}),256)]=SynapseXen_IIliiIliIl[SynapseXen_lillIIili(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[518845294],SynapseXen_iIilIIIlllIIiIiiI[340812766]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen best rerubi paste"SynapseXen_iIilIIIlllIIiIiiI[340812766]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(691533760,1908506525),SynapseXen_lillIIili(4047871218,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3326363781,3286290788,1940607795}return SynapseXen_iIilIIIlllIIiIiiI[340812766]end)(),262144),SynapseXen_ilIIiIIIllIli)]end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[185746739]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="this is so sad, alexa play ripull.mp4"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(245148454,2584298918)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1131944734,1131936676)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[185746739]=SynapseXen_lillIIili(SynapseXen_lillIIili(3408659963,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(4127481202,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3330981427,3460715256,3987404089,115879521,2586726668,2035975475,829488514,4143578923,1128989440,170127345}return SynapseXen_iIilIIIlllIIiIiiI[185746739]end)(10621,14115,"ilIllIlIII",225,8841,9307))then local SynapseXen_lllIiIiliiIIIIIil=SynapseXen_iIllilIiiIiIiIIIii[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[518845294],SynapseXen_iIilIIIlllIIiIiiI[2210192898]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="xen detects custom getfenv"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(449103776,4122622702)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3265338783,3265334156)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2210192898]=SynapseXen_lillIIili(SynapseXen_lillIIili(1104531221,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2252006083,SynapseXen_IIliiIliIl[5]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{505238930,2758057635,1140611233,4043760381,1702200857,896748878,1003852878,3254846394}return SynapseXen_iIilIIIlllIIiIiiI[2210192898]end)({},2968,{},"liiiIII",{},{}),262144)]local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;local SynapseXen_lIlili;local SynapseXen_IIiIIiIiiiiiiilIIii;if SynapseXen_lllIiIiliiIIIIIil[982698750]~=0 then SynapseXen_lIlili={}SynapseXen_IIiIIiIiiiiiiilIIii=setmetatable({},{__index=function(SynapseXen_lllllIIIiI,SynapseXen_iIIlilllIIi)local SynapseXen_IliIllIIiIIllII=SynapseXen_lIlili[SynapseXen_iIIlilllIIi]return SynapseXen_IliIllIIiIIllII[1][SynapseXen_IliIllIIiIIllII[2]]end,__newindex=function(SynapseXen_lllllIIIiI,SynapseXen_iIIlilllIIi,SynapseXen_lIIilliiiIIl)local SynapseXen_IliIllIIiIIllII=SynapseXen_lIlili[SynapseXen_iIIlilllIIi]SynapseXen_IliIllIIiIIllII[1][SynapseXen_IliIllIIiIIllII[2]]=SynapseXen_lIIilliiiIIl end})for SynapseXen_llllIiIIlliIilIlIl=1,SynapseXen_lllIiIiliiIIIIIil[982698750]do local SynapseXen_IilIiil=SynapseXen_IiiIiii[SynapseXen_iilliIiiiliillIIIllI]if SynapseXen_IilIiil[1902504146]==(SynapseXen_iIilIIIlllIIiIiiI[850224607]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi my 2.5mb script doesn't work with xen please help"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(679904342,3024402655)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(4282569902,4282623125)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[850224607]=SynapseXen_lillIIili(SynapseXen_lillIIili(321744706,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2158187034,SynapseXen_IIliiIliIl[11]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{130110110,3619705341}return SynapseXen_iIilIIIlllIIiIiiI[850224607]end)({},{},"iiil",5250,"iillIillIIliIi",{},5379,"iliiiIIliIl","lIIlliIllIIliIlIII","liIili"))then SynapseXen_lIlili[SynapseXen_llllIiIIlliIilIlIl-1]={SynapseXen_lliilillIl,SynapseXen_lillIIili(SynapseXen_IilIiil[31674126],SynapseXen_iIilIIIlllIIiIiiI[1124783093]or(function()local SynapseXen_IlIlllllIIIIlIilii="xen doesn't come with instance caching, sorry superskater"SynapseXen_iIilIIIlllIIiIiiI[1124783093]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(27982808,3974086151),SynapseXen_lillIIili(2046044,SynapseXen_IIliiIliIl[3]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1128941279,1753435619,3426622725,3075028745,1792634794,2177949522}return SynapseXen_iIilIIIlllIIiIiiI[1124783093]end)())}elseif SynapseXen_IilIiil[1902504146]==(SynapseXen_iIilIIIlllIIiIiiI[2855688064]or(function()local SynapseXen_IlIlllllIIIIlIilii="sponsored by ironbrew, jk xen is better"SynapseXen_iIilIIIlllIIiIiiI[2855688064]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2125024702,27583848),SynapseXen_lillIIili(3711361683,SynapseXen_IIliiIliIl[4]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2583821523}return SynapseXen_iIilIIIlllIIiIiiI[2855688064]end)())then SynapseXen_lIlili[SynapseXen_llllIiIIlliIilIlIl-1]={SynapseXen_lIlIiiIiIi,SynapseXen_lillIIili(SynapseXen_IilIiil[31674126],SynapseXen_iIilIIIlllIIiIiiI[3580195363]or(function()local SynapseXen_IlIlllllIIIIlIilii="skisploit is the superior obfuscator, clearly."SynapseXen_iIilIIIlllIIiIiiI[3580195363]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1435741229,3700070807),SynapseXen_lillIIili(283542476,SynapseXen_IIliiIliIl[6]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2635743001}return SynapseXen_iIilIIIlllIIiIiiI[3580195363]end)())}end;SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 end;SynapseXen_IilIIliiiliII[#SynapseXen_IilIIliiiliII+1]=SynapseXen_lIlili end;SynapseXen_lliilillIl[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[4268229530]or(function()local SynapseXen_IlIlllllIIIIlIilii="can we have an f in chat for ripull"SynapseXen_iIilIIIlllIIiIiiI[4268229530]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3949073557,4106912945),SynapseXen_lillIIili(3055211518,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1838134666,2797241552,3574957084}return SynapseXen_iIilIIIlllIIiIiiI[4268229530]end)(),256)]=SynapseXen_iliiiIllIiliiiI(SynapseXen_lllIiIiliiIIIIIil,SynapseXen_iiillIIIlIi,SynapseXen_IIiIIiIiiiiiiilIIii)elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3724519705]or(function()local SynapseXen_IlIlllllIIIIlIilii="inb4 posted on exploit reports section"SynapseXen_iIilIIIlllIIiIiiI[3724519705]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1220497714,1102045410),SynapseXen_lillIIili(2698166115,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3823207625,3762216720}return SynapseXen_iIilIIIlllIIiIiiI[3724519705]end)())then SynapseXen_ilIIiIIIllIli=SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[4263432476]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="my way to go against expwoiting is to have safety measuwes. i 1 wocawscwipt and onwy moduwes. hewe's how it wowks: this scwipt bewow stowes the moduwes in a tabwe fow each moduwe we send the wist with the moduwes and moduwe infowmation and use inyit a function in my moduwe that wiww stowe the info and aftew it has send to aww the moduwes it wiww dewete them. so whenyevew the cwient twies to hack they cant get the moduwes. onwy this peace of wocawscwipt."local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(254258454,1589279269)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1283588723,1283595680)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4263432476]=SynapseXen_lillIIili(SynapseXen_lillIIili(463583742,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3893060780,SynapseXen_IIliiIliIl[4]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3566371451,1724216092,574601064,4267456733,3512870450,569008975,424895593}return SynapseXen_iIilIIIlllIIiIiiI[4263432476]end)(7915,{},{},"IiiI",{},"iliIIl","liiIlII"),256)]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2397001969]or(function()local SynapseXen_IlIlllllIIIIlIilii="what are you trying to say? that fucking one dot + dot + dot + many dots is not adding adding 1 dot + dot and then adding all the dots together????"SynapseXen_iIilIIIlllIIiIiiI[2397001969]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3490931856,1932401406),SynapseXen_lillIIili(16901376,SynapseXen_IIliiIliIl[4]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3823151748,3637674999,2515343173,3678409867,407813225,3851658501,2022076928,372340199,2645111649}return SynapseXen_iIilIIIlllIIiIiiI[2397001969]end)())then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_lillIIili(SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[991069842]or(function()local SynapseXen_IlIlllllIIIIlIilii="https://twitter.com/Ripull_RBLX/status/1059334518581145603"SynapseXen_iIilIIIlllIIiIiiI[991069842]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2937374528,1690808131),SynapseXen_lillIIili(3528702417,SynapseXen_IIliiIliIl[10]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1307759604,620975304}return SynapseXen_iIilIIIlllIIiIiiI[991069842]end)(),256),SynapseXen_ilIIiIIIllIli)~=0;local SynapseXen_IIlill=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[2941932981]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="level 1 crook = luraph, level 100 boss = xen"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3979546062,3744941953)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2817128917,2817123694)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2941932981]=SynapseXen_lillIIili(SynapseXen_lillIIili(4256656460,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1712198787,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1124027665}return SynapseXen_iIilIIIlllIIiIiiI[2941932981]end)(5871,4286)),SynapseXen_ilIIiIIIllIli,512)local SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[3747975248]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="sometimes it be like that"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3478367805,1068169932)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2761337261,2761404249)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3747975248]=SynapseXen_lillIIili(SynapseXen_lillIIili(1111270663,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2589077317,SynapseXen_IIliiIliIl[5]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4281149359,4177838665}return SynapseXen_iIilIIIlllIIiIiiI[3747975248]end)(8239,{},11445),512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;if SynapseXen_IIlill<SynapseXen_lllil~=SynapseXen_lllIIiiIIIlIiIilii then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[571759888]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi xen doesn't work on sk8r please help"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1039541434,2529873645)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1366933515,1366939895)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[571759888]=SynapseXen_lillIIili(SynapseXen_lillIIili(2010681229,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2093755024,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2658363254,2688146088,1925363812,174348290}return SynapseXen_iIilIIIlllIIiIiiI[571759888]end)({}))then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3306826453]or(function()local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"SynapseXen_iIilIIIlllIIiIiiI[3306826453]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2966845895,2811732757),SynapseXen_lillIIili(3204261956,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2765635886,3062962255,2534064592,2626579238,2495973752,1111905447,637976218,888296673}return SynapseXen_iIilIIIlllIIiIiiI[3306826453]end)(),256)local SynapseXen_IIlill=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3233050271]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="epic gamer vision"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1742977386,2660661715)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(588152500,588207584)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3233050271]=SynapseXen_lillIIili(SynapseXen_lillIIili(3929861173,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3136304520,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1042019169,1713364601,3802338855,2249293969,1319005044,2660235727,3318850337,529694861,952922916,559907976}return SynapseXen_iIilIIIlllIIiIiiI[3233050271]end)("li","lilliiillIlliilIli",5830,{},1881,62,{},{},"I"),512)local SynapseXen_lliilillIl,SynapseXen_IIliilIIIliIIiliI=SynapseXen_IliliIlilllliIIlll,SynapseXen_liilIiiI;SynapseXen_lllilIiIill=SynapseXen_lllIIiiIIIlIiIilii-1;for SynapseXen_llllIiIIlliIilIlIl=SynapseXen_lllIIiiIIIlIiIilii,SynapseXen_lllIIiiIIIlIiIilii+(SynapseXen_IIlill>0 and SynapseXen_IIlill-1 or SynapseXen_IlliiilIIilIli)do SynapseXen_lliilillIl[SynapseXen_llllIiIIlliIilIlIl]=SynapseXen_IIliilIIIliIIiliI[SynapseXen_llllIiIIlliIilIlIl-SynapseXen_lllIIiiIIIlIiIilii]end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[1470589228]or(function()local SynapseXen_IlIlllllIIIIlIilii="thats how mafia works"SynapseXen_iIilIIIlllIIiIiiI[1470589228]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3687962295,4187329595),SynapseXen_lillIIili(2348524807,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4051482062,3240312872,3580862992,1421766586,3292097065}return SynapseXen_iIilIIIlllIIiIiiI[1470589228]end)())then local SynapseXen_lllil=SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[3256517115]or(function()local SynapseXen_IlIlllllIIIIlIilii="epic gamer vision"SynapseXen_iIilIIIlllIIiIiiI[3256517115]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(907037279,4153336600),SynapseXen_lillIIili(3909892363,SynapseXen_IIliiIliIl[5]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{883516762}return SynapseXen_iIilIIIlllIIiIiiI[3256517115]end)())local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3387102635]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="baby i just fell for uwu,,,,,, i wanna be with uwu!11!!"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3385753737,850469970)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1923028981,1923035956)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3387102635]=SynapseXen_lillIIili(SynapseXen_lillIIili(2693511574,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(349627000,SynapseXen_IIliiIliIl[2]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3609464111,1971423933,1462892655,1857117850,1138822157,3704956210,1710323307}return SynapseXen_iIilIIIlllIIiIiiI[3387102635]end)("li",{},"iiIIii",{},{}),256)]=SynapseXen_lliilillIl[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3073295919]or(function()local SynapseXen_IlIlllllIIIIlIilii="now with shitty xor string obfuscation"SynapseXen_iIilIIIlllIIiIiiI[3073295919]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1738453758,3656358790),SynapseXen_lillIIili(663260416,SynapseXen_IIliiIliIl[6]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3767105465,2145318903,2969619588}return SynapseXen_iIilIIIlllIIiIiiI[3073295919]end)())][SynapseXen_lllil]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[4112974965]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi xen doesn't work on sk8r please help"SynapseXen_iIilIIIlllIIiIiiI[4112974965]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1703548366,482264974),SynapseXen_lillIIili(3642536114,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4136099651,205277515,61143247,4272516003,618114322,2459165052}return SynapseXen_iIilIIIlllIIiIiiI[4112974965]end)())then local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;for SynapseXen_llllIiIIlliIilIlIl=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1762245251]or(function()local SynapseXen_IlIlllllIIIIlIilii="print(bytecode)"SynapseXen_iIilIIIlllIIiIiiI[1762245251]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2414310645,759173456),SynapseXen_lillIIili(2864118082,SynapseXen_IIliiIliIl[12]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4114536500,1968478769,1190877110,1438556347,3082873762,2149430889,350085903,3387045186}return SynapseXen_iIilIIIlllIIiIiiI[1762245251]end)()),SynapseXen_ilIIiIIIllIli,256),SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[107157657]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="this is a christian obfuscator, no cursing allowed in our scripts"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2999386983,3053263823)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3966009444,3966081325)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[107157657]=SynapseXen_lillIIili(SynapseXen_lillIIili(112651250,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2712040771,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1746880893,2123649085}return SynapseXen_iIilIIIlllIIiIiiI[107157657]end)("ilIiIilIlliiliIIl",{},{},14855,{},"i"),512)do SynapseXen_lliilillIl[SynapseXen_llllIiIIlliIilIlIl]=nil end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[2849142947]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="so if you'we nyot awawe of expwoiting by this point, you've pwobabwy been wiving undew a wock that the pionyeews used to wide fow miwes. wobwox is often seen as an expwoit-infested gwound by most fwom the suwface, awthough this isn't the case."local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3760542957,3672776847)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2026527117,2026584597)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2849142947]=SynapseXen_lillIIili(SynapseXen_lillIIili(2484316201,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(123920875,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3746198816,919089025,1646211836,1826023151,1493952422,1833762519,3249330014,424643384,3870666635,3904700363}return SynapseXen_iIilIIIlllIIiIiiI[2849142947]end)("iIlliII",{},3605,"iiilI","lliiilii"))then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+SynapseXen_iIIiillIiIIililIiiI[1050029882]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[1950146811]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="wally bad bird"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(933581639,1051343947)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1053428445,1053452426)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1950146811]=SynapseXen_lillIIili(SynapseXen_lillIIili(2147930552,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(548902678,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4101797929,1774570416,952759245,4014871170,3148648756,1954413750,2552319068,860402982}return SynapseXen_iIilIIIlllIIiIiiI[1950146811]end)("iilliIiililll",{},{}))then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1749618922]or(function()local SynapseXen_IlIlllllIIIIlIilii="inb4 posted on exploit reports section"SynapseXen_iIilIIIlllIIiIiiI[1749618922]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(759989537,83491605),SynapseXen_lillIIili(2148269523,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1614823055}return SynapseXen_iIilIIIlllIIiIiiI[1749618922]end)()),SynapseXen_ilIIiIIIllIli,256)~=0;local SynapseXen_IIlill=SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3708284236]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="xen doesn't come with instance caching, sorry superskater"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2743708921,1268609585)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3770822533,3770827770)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3708284236]=SynapseXen_lillIIili(SynapseXen_lillIIili(3261644362,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2210592095,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{2562407092,2336888471,2327589269,136376329}return SynapseXen_iIilIIIlllIIiIiiI[3708284236]end)("IIIIIllIIillllilll","lIilIIlIiiiiIlIIill",{},3296,{}))local SynapseXen_lllil=SynapseXen_iiIIlliIIIiliil(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[2379767592]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi xen crashes on my axon paste plz help"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2742007599,893438041)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2583033765,2583029384)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2379767592]=SynapseXen_lillIIili(SynapseXen_lillIIili(1918535990,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2837465477,SynapseXen_IIliiIliIl[9]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{2397854344,2988331668,3867117384,1773142231,742381117,3569487005}return SynapseXen_iIilIIIlllIIiIiiI[2379767592]end)("lIiIllliillliilIl","lllilllliIllIili","liiIl",{},{}),512),SynapseXen_ilIIiIIIllIli,512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;if SynapseXen_IIlill<=SynapseXen_lllil~=SynapseXen_lllIIiiIIIlIiIilii then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[1914006819]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="what are you trying to say? that fucking one dot + dot + dot + many dots is not adding adding 1 dot + dot and then adding all the dots together????"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3965014744,3584043923)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1393786358,1393844795)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1914006819]=SynapseXen_lillIIili(SynapseXen_lillIIili(3852470104,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1978178812,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{751339829,3553104005,598530110,2337167700,1459412150,2415423111}return SynapseXen_iIilIIIlllIIiIiiI[1914006819]end)("ilIIIlIIillillilllI"))then SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2063752794]or(function()local SynapseXen_IlIlllllIIIIlIilii="my way to go against expwoiting is to have safety measuwes. i 1 wocawscwipt and onwy moduwes. hewe's how it wowks: this scwipt bewow stowes the moduwes in a tabwe fow each moduwe we send the wist with the moduwes and moduwe infowmation and use inyit a function in my moduwe that wiww stowe the info and aftew it has send to aww the moduwes it wiww dewete them. so whenyevew the cwient twies to hack they cant get the moduwes. onwy this peace of wocawscwipt."SynapseXen_iIilIIIlllIIiIiiI[2063752794]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3915812109,1166532143),SynapseXen_lillIIili(2745355678,SynapseXen_IIliiIliIl[11]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2592495381,189363932,3260510402,4282285316,1399585652,3706097921,3250454677}return SynapseXen_iIilIIIlllIIiIiiI[2063752794]end)(),256)]=not SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[2151295232]or(function()local SynapseXen_IlIlllllIIIIlIilii="this is so sad, alexa play ripull.mp4"SynapseXen_iIilIIIlllIIiIiiI[2151295232]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2611177796,3190645646),SynapseXen_lillIIili(3369928768,SynapseXen_IIliiIliIl[3]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1667490949,3157511712,3334813471}return SynapseXen_iIilIIIlllIIiIiiI[2151295232]end)())]elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[1932937074]or(function()local SynapseXen_IlIlllllIIIIlIilii="double-header fair! this rationalization has a overenthusiastically anticheat! you will get nonpermissible for exploiting!"SynapseXen_iIilIIIlllIIiIiiI[1932937074]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(4249917206,3223517244),SynapseXen_lillIIili(2496451550,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3199305194,1611985699,764837406,586302449,3664328266,415743410,3482936446,2462962140,3537354557}return SynapseXen_iIilIIIlllIIiIiiI[1932937074]end)())then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2362686203]or(function()local SynapseXen_IlIlllllIIIIlIilii="my way to go against expwoiting is to have safety measuwes. i 1 wocawscwipt and onwy moduwes. hewe's how it wowks: this scwipt bewow stowes the moduwes in a tabwe fow each moduwe we send the wist with the moduwes and moduwe infowmation and use inyit a function in my moduwe that wiww stowe the info and aftew it has send to aww the moduwes it wiww dewete them. so whenyevew the cwient twies to hack they cant get the moduwes. onwy this peace of wocawscwipt."SynapseXen_iIilIIIlllIIiIiiI[2362686203]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(826762706,2609457382),SynapseXen_lillIIili(58567480,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3108205478,648071478,3792875852,2899683380,2026485168,3121027297,354787231,2812394854,2929654732,2180956010}return SynapseXen_iIilIIIlllIIiIiiI[2362686203]end)(),256),SynapseXen_ilIIiIIIllIli,256)local SynapseXen_IIlill=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3880811142]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="i put more time into this shitty list of dead memes then i did into the obfuscator itself"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(58192181,3984411172)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2674124550,2674115059)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3880811142]=SynapseXen_lillIIili(SynapseXen_lillIIili(1987204514,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(46421420,SynapseXen_IIliiIliIl[8]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{329910666,3038561987}return SynapseXen_iIilIIIlllIIiIiiI[3880811142]end)({}),512)local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;local SynapseXen_iIlllIilIIII,SynapseXen_illlIlIIIIIl;local SynapseXen_lIiIiliIiliiillIll;if SynapseXen_IIlill==1 then return elseif SynapseXen_IIlill==0 then SynapseXen_lIiIiliIiliiillIll=SynapseXen_lllilIiIill else SynapseXen_lIiIiliIiliiillIll=SynapseXen_lllIIiiIIIlIiIilii+SynapseXen_IIlill-2 end;SynapseXen_illlIlIIIIIl={}SynapseXen_iIlllIilIIII=0;for SynapseXen_llllIiIIlliIilIlIl=SynapseXen_lllIIiiIIIlIiIilii,SynapseXen_lIiIiliIiliiillIll do SynapseXen_iIlllIilIIII=SynapseXen_iIlllIilIIII+1;SynapseXen_illlIlIIIIIl[SynapseXen_iIlllIilIIII]=SynapseXen_lliilillIl[SynapseXen_llllIiIIlliIilIlIl]end;return SynapseXen_illlIlIIIIIl,SynapseXen_iIlllIilIIII elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[3310639834]or(function()local SynapseXen_IlIlllllIIIIlIilii="aspect network better obfuscator"SynapseXen_iIilIIIlllIIiIiiI[3310639834]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(1380206128,2397054583),SynapseXen_lillIIili(1175945315,SynapseXen_IIliiIliIl[8]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1409492963,753346180,3345752750,1577485556,960513600,4008949528,1632888873,3548351563,489013377,3312969157}return SynapseXen_iIilIIIlllIIiIiiI[3310639834]end)())then local SynapseXen_IIlill=SynapseXen_IliliIlilllliIIlll[SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[817622652]or(function()local SynapseXen_IlIlllllIIIIlIilii="print(bytecode)"SynapseXen_iIilIIIlllIIiIiiI[817622652]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(307590017,1353620065),SynapseXen_lillIIili(3770708343,SynapseXen_IIliiIliIl[4]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{419999639}return SynapseXen_iIilIIIlllIIiIiiI[817622652]end)(),512)]if not not SynapseXen_IIlill==(SynapseXen_lillIIili(SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[861585713]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="this is a christian obfuscator, no cursing allowed in our scripts"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(478172542,1038942354)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1097192040,1097190366)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[861585713]=SynapseXen_lillIIili(SynapseXen_lillIIili(2335438865,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(866592071,SynapseXen_IIliiIliIl[6]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1803516,781091480,1392758876,1566005029,2732403486,1948890459}return SynapseXen_iIilIIIlllIIiIiiI[861585713]end)({},{},"IiliIilIIIi",{},{},10912,"liIiIIilIIllI"),512),SynapseXen_ilIIiIIIllIli)==0)then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1 else SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[1157053867]or(function()local SynapseXen_IlIlllllIIIIlIilii="thats how mafia works"SynapseXen_iIilIIIlllIIiIiiI[1157053867]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(3153238270,1939183200),SynapseXen_lillIIili(3341449765,SynapseXen_IIliiIliIl[11]))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4291585478,649789820,2766051489}return SynapseXen_iIilIIIlllIIiIiiI[1157053867]end)(),256)]=SynapseXen_IIlill end elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[4030151273]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi my 2.5mb script doesn't work with xen please help"SynapseXen_iIilIIIlllIIiIiiI[4030151273]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(2370849318,1706000866),SynapseXen_lillIIili(95049263,SynapseXen_IIliiIliIl[3]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{412467578,2976784186,4213005441,3219785210,2834061951,173765433,2179268621}return SynapseXen_iIilIIIlllIIiIiiI[4030151273]end)())then local SynapseXen_IIlill,SynapseXen_lllil=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[927314387]or(function()local SynapseXen_IlIlllllIIIIlIilii="epic gamer vision"SynapseXen_iIilIIIlllIIiIiiI[927314387]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(807802397,2274683871),SynapseXen_lillIIili(504241661,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{4287587948,811356152,2249641043,3349513700,1457561955,2641009942}return SynapseXen_iIilIIIlllIIiIiiI[927314387]end)(),512),SynapseXen_ilIIiIIIllIli,512),SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[659900338]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="baby i just fell for uwu,,,,,, i wanna be with uwu!11!!"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2602039735,3285084897)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3019420033,3019479348)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[659900338]=SynapseXen_lillIIili(SynapseXen_lillIIili(1261583530,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3126415870,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3855480927,1587152270,330628007,2063308755}return SynapseXen_iIilIIIlllIIiIiiI[659900338]end)({},"IIII",{},"IIIiil","iliillilIIlI",{},{}))local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_IIlill>255 then SynapseXen_IIlill=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_IIlill-256]else SynapseXen_IIlill=SynapseXen_lliilillIl[SynapseXen_IIlill]end;if SynapseXen_lllil>255 then SynapseXen_lllil=SynapseXen_IIilIIiiIIIiiliIil[SynapseXen_lllil-256]else SynapseXen_lllil=SynapseXen_lliilillIl[SynapseXen_lllil]end;SynapseXen_lliilillIl[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[2674217529]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi devforum"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1246190301,2579588716)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1159290788,1159354561)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[2674217529]=SynapseXen_lillIIili(SynapseXen_lillIIili(3471940042,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3032837365,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1974699964,1764810507,1187749265,471585116,552842601}return SynapseXen_iIilIIIlllIIiIiiI[2674217529]end)({},12456,"IliiIlIiII",{},"IiiI"))][SynapseXen_IIlill]=SynapseXen_lllil elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[4137618607]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="now comes with a free n word pass"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1794406332,2321094792)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2952474109,2952473113)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4137618607]=SynapseXen_lillIIili(SynapseXen_lillIIili(1276689733,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(211665552,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{2944849226,231444223,3526260410}return SynapseXen_iIilIIIlllIIiIiiI[4137618607]end)("IllIliI",{},"IiiIIiii",{},{},5240,"IlIliIiilliIIi","IllliiIl",6958,"ilillIlI"))then SynapseXen_IliliIlilllliIIlll[SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3449421571]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="luraph better then xen bros :pensive:"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(1795003739,730360698)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1437095273,1437144580)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3449421571]=SynapseXen_lillIIili(SynapseXen_lillIIili(4179478510,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3072823438,SynapseXen_IIliiIliIl[11]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3495904001,3431603746,1434099189,745289844,2816713576,148556184,551112015,642319023}return SynapseXen_iIilIIIlllIIiIiiI[3449421571]end)({}),256)]={}elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[4156164161]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="hi xen crashes on my axon paste plz help"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3533602301,2364773069)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(1431636464,1431636796)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[4156164161]=SynapseXen_lillIIili(SynapseXen_lillIIili(48847484,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(4114143462,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1517631912,447543520,3894379304,3844495265,3589407916}return SynapseXen_iIilIIIlllIIiIiiI[4156164161]end)({},"llI","IlI",2878,"IIIiIiiliIi","lIlilllIiilllIlilll",{},1941,9947))then local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;local SynapseXen_IIlill=SynapseXen_IiiilIIIIiIIIlI(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[3286099876]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="what are you trying to say? that fucking one dot + dot + dot + many dots is not adding adding 1 dot + dot and then adding all the dots together????"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(476468903,3986716745)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3327403517,3327407270)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3286099876]=SynapseXen_lillIIili(SynapseXen_lillIIili(321193460,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(1120624031,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3876391237,3056048917,2997107476,588333636,4237181954,2514376226,2746281112}return SynapseXen_iIilIIIlllIIiIiiI[3286099876]end)(9286,"IllIIIlIiIilII","iiIIiIIIIIiIIill","liiiiIiI","IliiIIIiilIIiIl",{},"IIiIIIiil","liIllIIIliliIiiiii"),512)local SynapseXen_ilIlIII=SynapseXen_lliilillIl[SynapseXen_IIlill]for SynapseXen_llllIiIIlliIilIlIl=SynapseXen_IIlill+1,SynapseXen_iiIIlliIIIiliil(SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[84666744]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="yed"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(63324173,2567780398)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2712669476,2712671297)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[84666744]=SynapseXen_lillIIili(SynapseXen_lillIIili(582782751,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(298560836,SynapseXen_IliIIlIIlIlIlI))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{1287849774,481540864,2582223661,1590952752}return SynapseXen_iIilIIIlllIIiIiiI[84666744]end)({},4196,{},"IlIiIi",603),512),SynapseXen_ilIIiIIIllIli,512)do SynapseXen_ilIlIII=SynapseXen_ilIlIII..SynapseXen_lliilillIl[SynapseXen_llllIiIIlliIilIlIl]end;SynapseXen_IliliIlilllliIIlll[SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[655723646]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="can we have an f in chat for ripull"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(2104199711,435769096)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(963786704,963778951)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[655723646]=SynapseXen_lillIIili(SynapseXen_lillIIili(1396393776,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(954342646,SynapseXen_IIliiIliIl[11]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{1836690924,3515396440,3507070521,196345698,3278684494,3429317630,1928244737}return SynapseXen_iIilIIIlllIIiIiiI[655723646]end)("IiilliliIIlIIlIiiil","IIIIiiilIi","iiIilIlIlIiIIlliii",2505))]=SynapseXen_ilIlIII elseif SynapseXen_IIllilI==(SynapseXen_iIilIIIlllIIiIiiI[1633893892]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="inb4 posted on exploit reports section"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(3052696128,1880832186)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(245480221,245537207)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[1633893892]=SynapseXen_lillIIili(SynapseXen_lillIIili(2531024936,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(3194402741,SynapseXen_IIliiIliIl[3]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3802628430,1772593878,2142121166,1566934597,3592204892}return SynapseXen_iIilIIIlllIIiIiiI[1633893892]end)({},{},{},11649))then local SynapseXen_lllIIiiIIIlIiIilii=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[1997878473],SynapseXen_iIilIIIlllIIiIiiI[3162216979]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="baby i just fell for uwu,,,,,, i wanna be with uwu!11!!"local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(4072294189,1731619382)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(3041938210,3042009391)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil-SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3162216979]=SynapseXen_lillIIili(SynapseXen_lillIIili(3087720880,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2376996737,SynapseXen_IIliiIliIl[7]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-SynapseXen_IIllilI-#{3574969592,811265706,747715662}return SynapseXen_iIilIIIlllIIiIiiI[3162216979]end)({},"lililiiiIii",3879,1394),256)local SynapseXen_IIlill=SynapseXen_iiIIlliIIIiliil(SynapseXen_iIIiillIiIIililIiiI[31674126],SynapseXen_iIilIIIlllIIiIiiI[2006539795]or(function()local SynapseXen_IlIlllllIIIIlIilii="hi devforum"SynapseXen_iIilIIIlllIIiIiiI[2006539795]=SynapseXen_lillIIili(SynapseXen_IIiIllllliI(4163635175,1897977643),SynapseXen_lillIIili(548406073,SynapseXen_IliIIlIIlIlIlI))-SynapseXen_IIllilI-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{3445172065,1319985767,3975678498,2802133078,3893372487,80865164,408830403,2786726630,866063998}return SynapseXen_iIilIIIlllIIiIiiI[2006539795]end)(),512)local SynapseXen_lllil=SynapseXen_lillIIili(SynapseXen_iIIiillIiIIililIiiI[233588846],SynapseXen_iIilIIIlllIIiIiiI[3481987479]or(function(...)local SynapseXen_IlIlllllIIIIlIilii="so if you'we nyot awawe of expwoiting by this point, you've pwobabwy been wiving undew a wock that the pionyeews used to wide fow miwes. wobwox is often seen as an expwoit-infested gwound by most fwom the suwface, awthough this isn't the case."local SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIiIllllliI(702794506,2364179599)local SynapseXen_iliiIIIIiiiiiI={...}for SynapseXen_IIillIiiiiIi,SynapseXen_Iilii in pairs(SynapseXen_iliiIIIIiiiiiI)do local SynapseXen_IliIlIIIlIIllIlI;local SynapseXen_iiIIIiIliiliiIIl=type(SynapseXen_Iilii)if SynapseXen_iiIIIiIliiliiIIl=="number"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii elseif SynapseXen_iiIIIiIliiliiIIl=="string"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_Iilii:len()elseif SynapseXen_iiIIIiIliiliiIIl=="table"then SynapseXen_IliIlIIIlIIllIlI=SynapseXen_IIiIllllliI(2506320214,2506375217)end;SynapseXen_IIIIliliiiiIiiil=SynapseXen_IIIIliliiiiIiiil+SynapseXen_IliIlIIIlIIllIlI end;SynapseXen_iIilIIIlllIIiIiiI[3481987479]=SynapseXen_lillIIili(SynapseXen_lillIIili(3408538124,SynapseXen_IIIIliliiiiIiiil),SynapseXen_lillIIili(2007370375,SynapseXen_IIliiIliIl[10]))-string.len(SynapseXen_IlIlllllIIIIlIilii)-#{145467582,3560106105,1044477835,2077391817,102797013,2138533319,2010636405,3523148384,3678039188}return SynapseXen_iIilIIIlllIIiIiiI[3481987479]end)(10417,5510,{},{},"IllilIiIiII"))local SynapseXen_lliilillIl=SynapseXen_IliliIlilllliIIlll;if SynapseXen_lllil==0 then SynapseXen_iilliIiiiliillIIIllI=SynapseXen_iilliIiiiliillIIIllI+1;SynapseXen_lllil=SynapseXen_IiiIiii[SynapseXen_iilliIiiiliillIIIllI][638088808]end;local SynapseXen_lIIliiI=(SynapseXen_lllil-1)*50;local SynapseXen_lIiIiIill=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii]if SynapseXen_IIlill==0 then SynapseXen_IIlill=SynapseXen_lllilIiIill-SynapseXen_lllIIiiIIIlIiIilii end;for SynapseXen_llllIiIIlliIilIlIl=1,SynapseXen_IIlill do SynapseXen_lIiIiIill[SynapseXen_lIIliiI+SynapseXen_llllIiIIlliIilIlIl]=SynapseXen_lliilillIl[SynapseXen_lllIIiiIIIlIiIilii+SynapseXen_llllIiIIlliIilIlIl]end end end end;local SynapseXen_ilIlliiIIlIil={...}for SynapseXen_llllIiIIlliIilIlIl=0,SynapseXen_IlliiilIIilIli do if SynapseXen_llllIiIIlliIilIlIl>=SynapseXen_lIiIlIIilIIilililll[1481819974]then SynapseXen_liilIiiI[SynapseXen_llllIiIIlliIilIlIl-SynapseXen_lIiIlIIilIIilililll[1481819974]]=SynapseXen_ilIlliiIIlIil[SynapseXen_llllIiIIlliIilIlIl+1]else SynapseXen_IliliIlilllliIIlll[SynapseXen_llllIiIIlliIilIlIl]=SynapseXen_ilIlliiIIlIil[SynapseXen_llllIiIIlliIilIlIl+1]end end;local SynapseXen_IIlill,SynapseXen_lllil=SynapseXen_iilIIllIlIiIlilIill()if SynapseXen_IIlill and SynapseXen_lllil>0 then return unpack(SynapseXen_IIlill,1,SynapseXen_lllil)end;return end end;local function SynapseXen_iiIlIlIiIliiiiiii(SynapseXen_iiIIIlIl,SynapseXen_iiillIIIlIi)local SynapseXen_iiilllll=SynapseXen_IlliI(SynapseXen_iiIIIlIl)return SynapseXen_iliiiIllIiliiiI(SynapseXen_iiilllll,SynapseXen_iiillIIIlIi or getfenv(0)),SynapseXen_iiilllll end;return SynapseXen_iiIlIlIiIliiiiiii(SynapseXen_ilIIl("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"),getfenv())()
end
local function func_glow()
local mouse = game:GetService('Players').LocalPlayer:GetMouse()
tool = Instance.new("Tool")
tool.RequiresHandle = false
tool.Name = "Glow"
local function getpart()
local part = mouse.Target
if not part:IsDescendantOf(game.Workspace.Trees) and not part:IsDescendantOf(game.Workspace.Map) and not part:IsDescendantOf(game.Workspace.BuildZones) and not part:IsDescendantOf(game.Workspace.TrashCans) then return part end
end
tool.Activated:connect(function()
local part = getpart()
local A_2 = CFrame.new(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
local A_3 = Vector3.new(0, 0, 0)
local A_4 = 1
local Event = game:GetService("ReplicatedStorage").Remotes.ClientCutProcess
Event:FireServer(part, A_2, A_3, A_4)
end)
tool.Parent = game.Players.LocalPlayer.Backpack
end
local weldHingePerfectlyWindow

local function func_weldHinge()
	if weldHingePerfectlyWindow and not weldHingePerfectlyWindow:IsClosed() then return end
	weldHingePerfectlyWindow = WeldHingeWindow(screenGui)
end

local StrongGrabWindow = MenuWindow:__Subclass('StrongGrabWindow') do
	function StrongGrabWindow:__SuperArgs(parent)
		return 'Strong Grab', parent
	end
	
	function StrongGrabWindow:__Construct()
		self:Padding(8, nil)
		
		self.multiplierFrame = self:MakeObject('Frame')
		self.multiplierFrame.BackgroundTransparency = 1
		
		self.multiplierLabel = makeGuiObject('TextLabel')
		self.multiplierLabel.Text = 'Multiplier'
		self.multiplierLabel.Size = UDim2.new(0.4, -4, 1, 0)
		self.multiplierLabel.TextXAlignment = Enum.TextXAlignment.Left
		self.multiplierLabel.Parent = self.multiplierFrame
		
		self.multiplierBox = makeGuiObject('TextBox')
		self.multiplierBox.Text = '5'
		self.multiplierBox.Size = UDim2.new(0.6, -4, 1, 0)
		self.multiplierBox.Position = UDim2.new(0.4, 4, 0, 0)
		self.multiplierBox.Parent = self.multiplierFrame
		self.numbersOnlyMultiplier = numbersOnly(self.multiplierBox)
		self.numbersOnlyMultiplier:SetDecimalsAllowed(true)
		
		self.help = self:MakeLabel('Your grab strength will be multiplied by this amount.')
		self.help.TextWrapped = true
		self.help.Size = UDim2.new(1, 0, 0, 36)
		
		self:ResizeContent()
		self:Height(self:ScrollHeight())
		
		self.bodyMovers = MwsUtils:GetObjectManipulationBodyMovers()
		
		self:Apply()
		
		self.conn = s.RunService.RenderStepped:Connect(function()
			self:Apply()
		end)
	end
	
	function StrongGrabWindow:Apply(multiplier)
		local multiplier = multiplier or self.numbersOnlyMultiplier:GetNumber(1)
		
		if not self.init then
			self.originalBpD = self.bodyMovers.BodyPosition.D
			self.originalBpP = self.bodyMovers.BodyPosition.P
			self.originalBpMF = self.bodyMovers.BodyPosition.MaxForce
			self.originalBgD = self.bodyMovers.BodyGyro.D
			self.originalBgP = self.bodyMovers.BodyGyro.P
			self.originalBgMT = self.bodyMovers.BodyGyro.MaxTorque
			self.init = true
		end
		
		self.bodyMovers.BodyPosition.D = self.originalBpD * multiplier
		self.bodyMovers.BodyPosition.P = self.originalBpP * multiplier
		self.bodyMovers.BodyPosition.MaxForce = self.originalBpMF * multiplier
		self.bodyMovers.BodyGyro.D = self.originalBgD * multiplier
		self.bodyMovers.BodyGyro.P = self.originalBgP * multiplier
		self.bodyMovers.BodyGyro.MaxTorque = self.originalBgMT * multiplier
	end
	
	function StrongGrabWindow:Close()
		MenuWindow.Close(self)
		
		self:Apply(1)
		self.conn:Disconnect()
	end
end

local strongGrabWindow

local function func_strongGrab()
	if strongGrabWindow and not strongGrabWindow:IsClosed() then return end
	strongGrabWindow = StrongGrabWindow(screenGui)
end

local window = MenuWindow('MWS Toolkit Mod.', screenGui)

window:MakeLabel('Build Zone')
window:MakeBtn('Abandon Build Zone', func_abandon)
window:MakeBtn('TP to Build Zone', func_tpToBuildZone)
window:MakeBtn('Sell Scrap', func_sellScrap)
window:MakeLabel('Part Manipulation')
window:MakeBtn('Perfect Resize', func_perfectResize)
window:MakeBtn('Perfect Place', func_perfectPlace)
window:MakeBtn('Weld Parts', func_weldParts)
window:MakeBtn('Merge Items', func_mergeItems)
window:MakeBtn('Dupe Part', func_dupePart)
window:MakeBtn('Detach Part', func_detachPart)
window:MakeBtn('Decimate Creation', func_decimateCreation)
window:MakeLabel('Utility')
window:MakeBtn('Weld Hinge Perfectly', func_weldHinge)
window:MakeLabel('Tweaks')
window:MakeBtn('Weld Visor', func_swelds)
window:MakeBtn('Strong Grab', func_strongGrab)
window:MakeLabel('Mod')
window:MakeBtn('Part Transparency', func_tt)
window:MakeBtn('Glow', func_glow)
window:MakeBtn('Abandon/Claim tool', func_ac)
window:MakeBtn('Goodbye', func_goodbye)
window:MakeBtn('Fling Punch', func_fling)
window:MakeBtn('Human Car', func_Humancar)
window:MakeBtn('InstaDupe', func_fastdupe)
window:MakeBtn('InstaDelete', func_delete)
window:MakeBtn('Future Is Bright', func_FiB)
window:MakeBtn('Tp Tool', func_tptool)
window:MakeBtn('Whitelist', func_whitelist)
window:MakeBtn('SeatKiller', func_sk)
window:MakeLabel('by LoganDark#4357').TextXAlignment = Enum.TextXAlignment.Right

screenGui.Parent = Sirhurt and get_hidden_gui() or s.CoreGui
